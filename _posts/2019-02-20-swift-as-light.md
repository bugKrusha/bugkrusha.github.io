# Introduction
The Glowforge is the 3D Laser Printer that allows you to make things using like this wallet that I carry, my necklace and this coaster with the Swift logo. To make something you can start with something as simple as a picture or you can make sophisticated objects by creating designs in your favorite vector design tool. Ultimately, we send an SVG to the printer to create your design. The web has been supporting SVGs since 1998 which means there is a ton of support for it. However, on mobile, if support goes beyond the  displaying the whole svg, you will need complex custom tools to support your needs. In this talk, I would like to walk you through how I build our iOS apps to allow our users to print from their mobile devices. Today we will cover:
1. Image processing 
  1. Custom filters
  1. Flood filling algorithms.
1. Drawing Bezier paths
1. Transformation in SVGs

# Glowforge: Fundamentals
Here is the design for the coaster. The single _blue_ line will be cut but the printer. All images and filled shapes will be engraved. That is, non-white pixels in images and shapes with a fill will be engraved and unfilled shapes will be cut.

# Trace
One of my favorite features of the Glowforge is what we call Trace since it requires no knowledge of specialized design tools. That is, you can take a picture with your camera and we will trace it for you using the laser. You can also add cut lines to it with a single tap.

Just start with an image that you draw or downloaded. Let’s take a photo. Ouch, this photo now has an ugly shadow and all black pixels will be engraved. So we have to clean up all images that the user takes using a filter. First we grayscale the image using a luminance filter. This will give us a more uniform pixel representation of the image. Then we use a custom `CIFilter` to make the image solid black and white in real time. `CIFilter` requires  an input image and an output image. In our case the out image is the black and white image. The heart and soul of a custom filter is the color kernel. A color kernel is a GPU based image processing routine that processes only the color information in images. Here is what ours look like:

``` swift 
class ThresholdFilter: CIFilter {
    @objc var inputImage: CIImage?
    var thresholdValue: Double = 128.0
    
    private var colorKernel: CIColorKernel? {
        return CIColorKernel(source:
            """
            kernel vec4 threshold(sampler inputImage) {
                vec4 pixel;
                pixel = sample(inputImage, samplerCoord(inputImage));
                float gray = pixel.r;
                float threshold = \(thresholdValue) / 255.0;
                float correctedGray = (1.0 - pixel.a) + (pixel.a * gray);
                float thresholdGray = correctedGray >= threshold ? 1.0 : 0.0;
            
                return vec4(thresholdGray, thresholdGray, thresholdGray, 1.0);
            }
            """
        )
    }
    //
}
```

Let’s walk through it. First what is this weird string thingy? It is Core Image Kernel Language which is its own thing with its own set of quirks. Since this is a string, it is incredibly prone to errors so you have to be really careful. It’s a kernel function called `threshold` that takes a sampler object as a parameter. 

1. This sampler is responsible for accessing image data and passing it to the kernel routine. We declare a `vec4` as a pixel. A `vec4` is a four element vector that can conveniently store the red, green, blue, and alpha components of a pixel.
1. We convert our given threshold to a value between zero and one by dividing our constant by 255. Where does 255 come from? In a grayscale image, the pixel value is a single value representing the brightness of the pixel. This value is stored as an 8 bit integer that ranges from 0 - 255 with 0 taken as black and 255 taken as white. Everything in between are shades of gray.
1. This looks fancy but it just ensures the transparency in a pixel is corrected since we can’t engrave transparent pixels
If our corrected gray is greater than or equal to the threshold, we use 1 which makes the pixel white. If not, we use 0 which makes the pixel black.
1. We then return the vec4 with those pixel values we calculated along with a 100% opacity value.

# Flood Filling
Now our image is ready and can be sent to the printer in its current state. However, the user can decide add cut lines by outlining the entire image or a closed area within the image. Here is what this owl will look like when are down with it. We want to cut out that area around its eyes to make it look rather evil. So we tap those areas, you will see that we draw a line within that area which will tell the printer that there should be a cut there. So how do we achieve this? This is achieved by using a flood filling algorithm to find all the pixels enclosed within the outlining border. Then we find all the border pixels and determine if the hit the border on the top, bottom, left or right and we store that. We then unwind those pixels to build a bezierpath(needs diagram).

Flood filling is used to define a region in an image that has the same colors. To achieve this, we establish that pixels have neighbors, top, bottom, left, right. In our case, the user will tap a pixel within a region and we will remember the color of that pixel. Any pixel of the same color that can be reached by a neighborhood relationship are marked as being within the region. If a pixel of a different color is encountered, it is regarded as a boundary. This process starts when the user selects a pixel. We put all the pixels of the image in an array.

## Getting Pixels into an Array
An image or bitmap is a two-dimensional array of elements with each element representing the colors in a pixel. To simplify things, I convert the image to one dimensional array using a buffer as shown.

``` Swift
typealias PixelBuffer = UnsafeMutableBufferPointer<Pixel>
```
Once we have our array, we can begin the flood filling process. I don’t want to spend time going through the full flood filling algorithm since you could find interesting ones online but I would like to cover some bits I find interesting. Recall that we said color intensity is represented by an 8 bit integer so I define a Pixel as follows:

``` Swift
struct Pixel: Equatable {
    var red: UInt8
    var green: UInt8
    var blue: UInt8
    var alpha: UInt8
}
```

## Touching a Pixel
When a user touches a pixel, you can find the x, y coordinate of that pixel in two dimensional space.

``` Swift
func imageTapped(_ sender: UITapGestureRecognizer) {
    let point = sender.location(in: drawableImageView)
}
```
However, all of our operations are based on a one dimensional array. Given the `x`, `y` coordinate in 2D array, how do you find the corresponding index in a 1D array:

``` Swift
private func indexOf(x: Int, y: Int) -> Int? {
    return inBounds(x: x, y: y) ? y * width + x: nil
}

private func inBounds(x: Int, y: Int) -> Bool {
    return y >= 0 && y < height && x >= 0 && x < width
}
```

If the pixel is in bounds, you find the corresponding index by `y * width + x`. 
_Show diagram here_

Multiplying the y position by the width will wrap the index all the way around to get the right row in the 2D array. Adding `x` will get us to the right column.

When the user sees an image on their screen, that image was scaled to fit the view. The implication is that the coordinate they the touch doesn’t necessarily correspond to the coordinate in the actual image. So we must find that coordinate. 

_Show coordinate difference image_

``` Swift
func convert(point: CGPoint) -> CGPoint {
    let xScale = imageWidth / (imageViewWidth - xMargin) // 1.
    let yScale = imageHeight / (imageViewHeight - yMargin) // 2.
    let x = (coordinate.x - xMargin) * xScale // 3.
    let y = (imageViewBounds.maxY - coordinate.y - yMargin) * yScale // 4.
    
    return CGPoint(x: x, y: y) // 5. 
}
```

1.  We find the xScale by dividing the image width by the imageView width. We subtract the margin because an aspect fit image might be smaller than the image view. _Show aspectfit diagram_ 
1.  Similar to 1.
1.  To get adjusted `x`, we multiply the x value in the point we pass in by the `x` scale.
1.  Same as above.
1.  In `CoreGraphics`, origin is bottom left and and in `UIKit` it is top right so we make that conversion here. _show difference diagram_
*Comment out transformation code to show big cut line* :sweat_smile:

So now we are ready to touch our area of interest to see the outcome. This obviously now what we want. Can you tell what happened here? The path that was drawn was drawn to fit the original image size, which is correct because that is how you want it printed. However, we still need to show the user the path in the area the expected, the filled region. _Show correct version_. So we must add a transformation to the path to make it fit the image as the user expects. This requires a translation and a scale.

``` Swift
struct PathTransformation {
    let x: CGFloat
    let y: CGFloat
    let scale: CGFloat
}

var pathTransformation: PathTransformation  {
    let scale = self.aspectFitSize.width / self.imageSize.width
    let xTranslation = (imageViewWidth - aspectWidth) / 2.0
    let yTranslation = (imageViewHeight - aspectHeight) / 2.0
    
    return PathTransformation(x: xTranslation, y: yTranslation, scale: scale)
}
```

# Drawing the path
Drawing the path has a bit of indirection because while building this, I ran into issues with drawing directly on an image view. So I make it such that we have a `DrawableView` that is transparent over the the `DrawableImageView`. Here is what they look like:
## DrawableView
```Swift
class DrawableView: UIView {
    private let lineWidth: CGFloat = Constants.lineWidth
    private let color = Colors.teal
    
    var path =  UIBezierPath() {
        didSet { setNeedsDisplay() } // 1.
    }
    
    // 2.
    override func draw(_ rect: CGRect) {
        path.lineWidth = lineWidth
        color.set()
        path.stroke()
    }
}
```

1.  Whenever we set the path on the drawable view, we redraw the view.
1.  In `draw(_ rect: )`, we set the line width of the path and set the color and call `stroke()` on `path`.

## DrawableImageView

``` Swift
final class DrawableImageView: UIImageView {
    let drawableView = DrawableView() // 1.
    
    override func layoutSubviews() {
        super.layoutSubviews()

        //
        addSubview(drawableView // 2.
        drawableView.frame = self.bounds // 3.
    }
}
```

1.  Declarae drawable view
1.  Add the drawable view as a subview
1.  Set the frame of the drawable view

## SVG Transformation
The Glowforge uses SVGs for design files. They are lightweight and ubiquitous on the web. On mobile, if you want to do anything but display a whole svg, you will need a custom solution.

`Show a svg then focus on the group:`

When a user loads their design, we extract the groups from the svg and create individual views that the user can interact with. That way, they can move them around to exactly where they want their print to be on the material. _Show screenshot_ Importantly, the position of the view on the screen has to translate precisely to the position on the material. Mistakes are costly since we are dealing with a laser that consumes the material. Let’s see how we extract the groups from the SVG, draw them on screen and precisely monitor their transformations.

First, we reduce the paths into a single bezierpath like this: 
`Show reduce code here`

We extract the string under the d attribute for each path. That contains the information about how the path should be drawn. Using a complicated algorithm, we can convert this string to Bezierpath.
We then reduce these paths into a single path.
We the use the `boundingBox` property on the bezierpath to get a frame for positioning..
We then use this path to create a shape layer like so.

`Shape layer code here`:

Note that we also extract the image data along with its transform and store them. In the end we have a class that looks like this:

`Show drag group code here:`

``` Swift

```

Drawing and Position:
Now that we have the drag group, we need use it draw and position a drag group view. This view will hold a image view for the image and the layer. The frame of the drag group view itself is rather straightforward, being the smallest frame that could fit both the frame of the layer and the frame of the image view. Let’s first talk about the layer. If I just add this layer to the layer of the DragGroupView, we get something like this :sob:

`show image here:`

The layer is positioned outside the view. Any ideas? They issue is that the origin of the layer that we calculated with `let frame = path.boundingBox` is the position within the entire svg. Now we would like the position inside this drag group view. Therefore, we create a translation using the negative values of the origin and set the layer’s transform as that. This basically moves the layer to the parent view’s origin.

`ImageView`
The image view get’s a little more interesting. Recall that the image tag had a transform, x and y origin, length and width. That means we have to create the image view with the given frame, then apply the transformation to it. Let’s do that. Again, this looks crazy :sob:
Not only is the positioning wrong, but so is the size and the rotation. Why could this be? Well, in svgs, transformation occur around the origin unless otherwise defined. Here is what that looks like:

`Show diagram of square rotate here:`

However, in iOS, rotation occurs around the center so look like this:

`Show center rotation here:`


So how do we achieve this? Ideas? Well to move the transformation to the origin from the center, we just translate the view using the negative values of its current x/y origin. Then, and only then we apply the transformation using a contenation. Once the transformation is over, we must translate the view back to its origin using the values of the center. That code will look like this:

`show code here:`

``` Swift

```


Closing
Building an app for the Glowforge means that we have to take a lot of technology that have existing solutions or work default and make them work for us. Image processing is emphasized on mobile and we can leverage native APIs to get our desired results. Working with SVGs and other web technologies though might take you down a path of temporary insanity but continued discovery. I hope I was able to show you some cool stuff today :)
