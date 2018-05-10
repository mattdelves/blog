---
layout: post
title:  "Rounded corners using a bezier path"
date:   2018-05-10 16:00:00 +1000
categories: technical
---

So todays adventure involved trying to get a `UIView` to have a nice rounded corner on it. I had recently seen a [tweet](https://twitter.com/nathangitter/status/992842150500069376?s=21) showing the advantages of using a `UIBezierPath` for applying rounded corners to a view. The use case in the tweet was a `UIButton` but I thought it would be nice to see how I could use it for any kind of `UIView` and in particular for a `UICollectionViewCell` as is often done.

As expected, there were a few things learnt along the way which are the focus of this blog post. To begin, lets outline what I was trying to achieve.

![start_image](/images/rounded_corners_1.png)

Within the outside view (red) there's a view (white) which needs to have rounded corners.

So, being responsible software developers we want to use composition to say that a subclass of `UIView` has rounded corners. The first step in this is defining a protocol around the behaviour.

``` swift
protocol RoundedCorners {
  var radius: CGFloat { get set }
  func applyRoundedCorners()
}
```

The information required here is the radius for the view and a function that takes the radius and applies it to the view. This can be defined in a protocol extension. We limit the protocol extension to being for anything that is a subclass of `UIView`.

``` swift
extension RoundedCorners where Self: UIView {
  func applyRoundedCorners() {
    let bezierPath = UIBezierPath(roundedRect: bounds, cornerRadius: radius)

    let maskLayer = CAShapeLayer()
    maskLayer.path = bezierPath.cgPath
    layer.mask = maskLayer
  }
}
```

The next thing is when do we call this function. This is where problems start to creep in. If we aren't resizing the view then we are free to set the view in many places. But lets set out some problems which will cause things to not behave as we expect.

The first consideration is that the view is part of a `UICollectionViewCell`. Typically when creating a a cell subclass, I like to build it in its own Xib file and then load it from there. We're not going to step through the behaviour here so lets just assume that's the case. The first problem encountered is that the Xib defining the cell was of a different width / height to what ends up being in the app when it runs. So why is this a problem? lets take a look at how we're defining the path for the rounded corners. We're using a convenience initializer that takes a `CGRect` and a `CGFloat` and creates the path from that. This is a problem if we're adjusting the size of the cell. It will result in the path being incorrect and producing a visual bug. This can be addressed by applying the mask layer at a point after which the bounds have been calculated. The best place to do this is in `layoutSubviews()`. We know at that point that the bounds have been calculated correctly.

The first attempt at this didn't work as expected. There are a few reasons for this. First of which is that I hadn't defined the protocol exactly as above. I was passing in both the radius and the view to apply things to. Let's take a look at what the visual output was.

![incorrect_bounds](/images/rounded_corners_2.png)

The corners were only applied to part of the view. Even though I was calling it from `layoutSubviews()` it didn't have the correct frame. So what was going on? I was scratching my head a bit as things just weren't working. The problem is that the frame still wasn't exactly as I expected it when it was having the mask applied. The solution came in the form of a `UIView` subclass. Let's take a look at this.

``` swift
final class RoundedView: UIView {
  var radius: CGFloat = 12.0

  override func layoutSubviews() {
    super.layoutSubviews()

    applyRoundedCorners()
  }
}

extension RoundedView: RoundedCorners { }
```

This worked in part. When I scrolled the collection view, the corners were correctly rounded when the cells ended up being reused. Unfortunately they weren't correct when the cells are first displayed. Something just wasn't correct. The corners needed to be applied at the start, not just when the cells get reused. The answer comes in the form of calling `applyRoundedCorners()` in `awakeFromNib()`. This looks like the following.

``` swift
override func awakeFromNib() {
  super.awakeFromNib()

  applyRoundedCorners()
}
```

The result of this was amazingly smooth and visual appealing rounded corners on a view. Job well done.

![final_produce](/images/rounded_corners_3.png)

This was a good learning experience for me. Trying to get my head around the view lifecycle is a daunting thing. It's something I struggle with but the more I work with UI stuff, the better I understand it all. Hope you all find it just as interesting and informative.
