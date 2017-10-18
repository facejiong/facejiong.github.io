---
title: d3-color
date: 2017-10-19 03:55:05
tags: d3
categories: js
---
d3-color支持多种颜色格式，rgb(red,green,blue)，hsl(hue,saturation,lightness)，lab，hcl，cubehelix。
```
d3.color('steelblue') // At{r: 70, g: 130, b: 180, opacity: 1}
d3.rgb('steelblue') // At{r: 70, g: 130, b: 180, opacity: 1}
d3.hsl('steelblue') // Rt{h: 207.27272727272728, s: 0.44, l: 0.4901960784313726, opacity: 1}
d3.lab('steelblue') // Dt{l: 52.46551718768575, a: -4.0774710123572255, b: -32.19186122981343, opacity: 1}
d3.hcl('steelblue') // Ht{h: 262.78126775909277, c: 32.44906314974561, l: 52.46551718768575, opacity: 1}
d3.cubehelix('steelblue') // Vt{h: 202.84837896488932, s: 0.6273147230777709, l: 0.460784355920337, opacity: 1}

d3.color('steelblue').toString() // "rgb(70, 130, 180)"
```
rgb,hsl,lab,hcl,cubehelix继承自color
```
export default function(constructor, factory, prototype) {
  constructor.prototype = factory.prototype = prototype;
  prototype.constructor = constructor;
}
// 返回parent.prototype的copy对象，该对象同时继承definition的属性和方法
export function extend(parent, definition) {
  var prototype = Object.create(parent.prototype);
  for (var key in definition) prototype[key] = definition[key];
  return prototype;
}

export function Color() {}

export function rgb(r, g, b, opacity) {
  return arguments.length === 1 ? rgbConvert(r) : new Rgb(r, g, b, opacity == null ? 1 : opacity);
}

export function Rgb(r, g, b, opacity) {
  this.r = +r;
  this.g = +g;
  this.b = +b;
  this.opacity = +opacity;
}

define(Rgb, rgb, extend(Color, {
  brighter: function(k) {
    k = k == null ? brighter : Math.pow(brighter, k);
    return new Rgb(this.r * k, this.g * k, this.b * k, this.opacity);
  },
  darker: function(k) {
    k = k == null ? darker : Math.pow(darker, k);
    return new Rgb(this.r * k, this.g * k, this.b * k, this.opacity);
  },
  rgb: function() {
    return this;
  },
  // 判断颜色是否有效
  displayable: function() {
    return (0 <= this.r && this.r <= 255)
        && (0 <= this.g && this.g <= 255)
        && (0 <= this.b && this.b <= 255)
        && (0 <= this.opacity && this.opacity <= 1);
  },
  toString: function() {
    var a = this.opacity; a = isNaN(a) ? 1 : Math.max(0, Math.min(1, a));
    return (a === 1 ? "rgb(" : "rgba(")
        + Math.max(0, Math.min(255, Math.round(this.r) || 0)) + ", "
        + Math.max(0, Math.min(255, Math.round(this.g) || 0)) + ", "
        + Math.max(0, Math.min(255, Math.round(this.b) || 0))
        + (a === 1 ? ")" : ", " + a + ")");
  }
}));
```
