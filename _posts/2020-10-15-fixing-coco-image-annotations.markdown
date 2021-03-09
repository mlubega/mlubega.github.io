---
layout: post
title:  "Fixing COCO Image Annotations"
date:   2020-10-15 23:10:30 +0100
categories: jekyll update
tags: mask-rcnn computer-vision coco-annotations object-detection object-segmentation
---

Do you see what's wrong with this picture? 

<div style="text-align:center">
<img src="/images/via_wrong.png"/>
</div>
&nbsp;

It's a bit hard to interpret, but it's a grey-scale picture of the surface of a bee hive with multi-colored oval-shaped
annotations _supposedly_ indicating the positions of bees in the image. Unfortunately for
me, the annotations are incorrecly oriented. Although they are positioned over the bees, the oval annotations are only horizontal or vertical.

I encountered this problem when I was working on a summer project to see how well 
Facebook's [Mask R-CNN](https://github.com/matterport/Mask_RCNN) model for Object
Detection & Segmentation does at segmenting bees. 

I was using [VIA Annotator](http://www.robots.ox.ac.uk/~vgg/software/via/) to label my data, but rather than drawing polygon vertices around each bee, I decided to speed up the process by using using oval anotations instead. After all, bees are roughly oblong shapes, and an oval would neatly capture the centerpoint point and angular orientation which is all I really cared about. So after a week of painstakingly drawing oval shapes over bees, I had 20 annotated images.

The actual annotations themselves conformed to the [COCO Dataset Format](https://www.immersivelimit.com/tutorials/create-coco-annotations-from-scratch/#coco-dataset-format) for segmentation. This is a JSON file with the following fields:

```json
{
    "info": {...},
    "licenses": [...],
    "images": [...],
    "annotations": [...],
    "categories": [...],
    "segment_info": [...]
}
```
The `segmentation` field within the `annotations` item is a list of `(x,y)` vertices describing the shape, but just looking at the numbers didn't give me an indication of what's wrong.
```json
{
 "images": [
    {
      "id": 0,
      "width": 1024,
      "height": 1024,
      "file_name": "1.png",
      "license": 1,
      "date_captured": ""
    },
    ....
 ],
 "annotations": [
    {
      "id": 0,
      "image_id": "0",
      "category_id": 2,
      "segmentation": [ 61, 143, 144.307, 60.772, 145.605, 60.489, 146.882 ... ],
      "area": 900,
      "bbox": [ 31, 128, 30, 30],
      "iscrowd": 0
    },
    ...
  ],
}
```
Fortunately VIA's source code is a simple javascript executable. Although I'm more of a backend engineer, I've dabbled in frontend code in the past.
 Looking into the conversion function in source file, I found the root of the problem. 

```javascript
function via_region_shape_to_coco_annotation(shape_attributes) {
  // code
  
  case 'ellipse':
      // code

      var x = shape_attributes['cx'] + a * Math.cos(theta_radian);
      var y = shape_attributes['cy'] + b * Math.sin(theta_radian);

```

That feeling when something you learned in high school turns out to be super useful! These two lines are describing parametric equations to draw an circle or ellipse. The centerpoint is `(cx,cy)`, `a` is the x-radius and `b` is the y-radius. If `a == b` you get a circle and `a != b` you get an ellipse. Turned back into math equations: 

<div style="text-align:center">
<a href="https://www.codecogs.com/eqnedit.php?latex=x&space;=&space;r_x&plus;a\cos(\Theta&space;)" target="_blank"><img src="https://latex.codecogs.com/gif.latex?x&space;=&space;r_x&plus;a\cos(\Theta&space;)" title="x = r_x+a\cos(\Theta )" /></a><br/>
<a href="https://www.codecogs.com/eqnedit.php?latex=y=&space;r_y&plus;b\sin(\Theta&space;)" target="_blank"><img src="https://latex.codecogs.com/gif.latex?y=&space;r_y&plus;b\sin(\Theta&space;)" title="y= r_y+b\sin(\Theta )" /></a>
</div>
&nbsp;

The problem is these equations don't account for the angle of rotation of an ellipse. For that, a different set of parametric equations is needed where \beta is the angle of rotation:

<div style="text-align:center">
<a href="https://www.codecogs.com/eqnedit.php?latex=x&space;=&space;r_x&plus;a\cos(\Theta&space;)\cos(\beta&space;)-b\sin(\Theta)sin(\beta)" target="_blank"><img src="https://latex.codecogs.com/gif.latex?x&space;=&space;r_x&plus;a\cos(\Theta&space;)\cos(\beta&space;)-b\sin(\Theta)sin(\beta)" title="x = r_x+a\cos(\Theta )\cos(\beta )-b\sin(\Theta)sin(\beta)" /></a><br/>
<a href="https://www.codecogs.com/eqnedit.php?latex=y&space;=&space;r_y&plus;a\cos(\Theta&space;)\sin(\beta&space;)&plus;&space;b\sin(\Theta)\cos(\beta)" target="_blank"><img src="https://latex.codecogs.com/gif.latex?y&space;=&space;r_y&plus;a\cos(\Theta&space;)\sin(\beta&space;)&plus;&space;b\sin(\Theta)\cos(\beta)" title="y = r_y+a\cos(\Theta )\sin(\beta )+ b\sin(\Theta)\cos(\beta)" /></a>
</div>
&nbsp;

```javascript
function via_region_shape_to_coco_annotation(shape_attributes) {
  // code
  case 'ellipse':
    var a = shape_attributes['rx'];
    var b = shape_attributes['ry'];
    var theta_to_radian = Math.PI/180;
    var rotation = shape_attributes['theta']
    for ( var theta = 0; theta < 360; theta = theta + VIA_POLYGON_SEGMENT_SUBTENDED_ANGLE ) {
      var theta_radian = theta * theta_to_radian; 
      var x = shape_attributes['cx'] + ( a * Math.cos(theta_radian) * Math.cos(rotation) ) - ( b * Math.sin(theta_radian) * Math.sin(rotation));
      var y = shape_attributes['cy'] + ( a * Math.cos(theta_radian) * Math.sin(rotation) ) + ( b * Math.sin(theta_radian) * Math.cos(rotation));
      annotation['segmentation'].push( fixfloat(x), fixfloat(y) );
    }    
    annotation['bbox'] = polygon_to_bbox(annotation['segmentation']);
    annotation['area'] = annotation['bbox'][2] * annotation['bbox'][3];
    break;

```
Since VIA is a standalone executable, I simply modified the source file to use the new equations, regenerated the coco annotations file and viola! Problem solved!


<div style="text-align:center">
<img src="/images/via_right.png" />
</div>
&nbsp;

I created a GitLab [issue](https://gitlab.com/vgg/via/-/issues/215) raising this bug and the proposed fix, and it has since been incorporated into via-2, but ellipses have been deprecated in via-3. 

&nbsp;

#### Further Reading
---
[COCO Dataset Format](https://www.immersivelimit.com/tutorials/create-coco-annotations-from-scratch/#coco-dataset-format)  
[Rotating and Translating an Ellipse with Parametric Equations](https://www.youtube.com/watch?v=Xuj8gY6He5w)
