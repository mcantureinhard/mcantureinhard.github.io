---
layout: post
title:  "WiFi Access Point Mode Thermal Camera with interpolation"
date:   2018-08-26 21:00:00 +0200
categories: [image-processing,  microcontroller, sensor, esp8266, arduino, javascript]
---
The AMG8833 is a sensor by Panasonic that allows to measure temperatures of a surface. The area is broken down into 64 parts as an 8x8 matrix. This sensor allows for up to 10 thermal measurements per second. In normal use cases, this is just fine and can be used as is. For this project I used the Feather HUZZAH with ESP8266 and the IR Thermal Camera FeatherWing from Adafruit. If we want to build a thermal camera, 8x8 is not a high resolution. we can use interpolation to infer data points between known data points.

The simplest type of interpolation we could use is bilinear. The algorithm for calculating bilinear interpolation can be easily found. A simple javascript implementation is as follows:
{% highlight javascript %}
function interpolate_points(p00, p01, p10, p11, x, y){
  var a00 = p00;
  var a10 = p10 - p00;
  var a01 = p01 - p00;
  var a11 = p11 + p00 - (p10 + p01);
  var r = a00 + a10*x + a01*y + a11*x*y;
  return r;
}

function bilinear_resize(arr_in, w, h, mult){
  var nh = h * mult;
  var nw = w * mult;
  var out = new Array(nh*nw).fill(0);
  for(var i = 0; i < h; i++){
    for(var j = 0; j < w; j++){
      out[i*mult*nw + j*mult] = arr_in[i*w+j];
      for(var k = 0; k < mult && k < nh; k++){
        for(var l = 0; l < mult && l < nw; l++){
          var py = k/mult;
          var px = l/mult;
          var mi = 0;
          var mj = 0;
          if((i+1)>=h){
            mi = 1;
          }
          if((j + 1)>=w){
            mj = 1;
          }
          out[(i*mult+k) * nw + j*mult + l] = interpolate_points(
              data[i*w+j],
              data[i*w + j + 1 - mj],
              data[(i+1-mi) * w +j],
              data[(i+1-mi) * w + j + 1 - mj],
              py,
              px
          );
        }
      }
    }
  }
  return out;
}
{% endhighlight %}

Bilinear interpolation works well, but the end result can still be a bit rough. I read about bicubic interpolation and how it is used to resize images. In the case of bicubic interpolation 16 points are used instead of 4, as the slopes on x and y are also required. I decided to try bicubic interpolation out to see if there was any improvement. I implemented a quick version based on a common matrix based algorithm. I added some quick code to handle the border. There are different ways to set values for imaginary exterior points. I wanted a quick test, so I decided to set them to zero. This will result in bad data near the border. The implementation is as follows: 

{% highlight javascript %}
function cubic_value(arr, x, y, p, h, w){
  var temp = new Array(4);
  temp[0] = y - 1 >= 0 ? arr[(y - 1) * w + x] : 0;
  temp[1] = y >= 0 && y <= h ? arr[y * w + x] : 0;
  temp[2] = y + 1 <= h ? arr[(y + 1) * w + x] : 0;
  temp[3] = y + 2 <= h ? arr[(y + 2) * w + x] : 0;
  return temp[1] + 0.5 * p*(temp[2] - temp[0] + p*(2.0*temp[0] - 5.0*temp[1] + 4.0*temp[2] - temp[3] + p*(3.0*(temp[1] - temp[2]) + temp[3] - temp[0])));
}

function bicubic_value(arr, y, x, py, px, h, w){
  var temp = new Array(4);
  temp[0] = x - 1 >= 0 ? cubic_value(arr, x-1, y, py, h, w) : 0;
  temp[1] = x >= 0 && x <= w ? cubic_value(arr, x, y, py, h, w) : 0;
  temp[2] = x + 1 <= w ? cubic_value(arr, x + 1, y, py, h, w) : 0;
  temp[3] = x + 2 <= w ? cubic_value(arr, x + 2, y, py, h, w) : 0;
  return cubic_value(temp, 0, 1, px, 4, 1);
}

function bicubic_resize(arr_in, w, h, mult){
    var min = arr_in[0];
    var max = arr_in[0];
    var n_w = w * mult;
    var n_h = h * mult;
    var out = new Array(n_w * n_h).fill(0);
    for(var y = 0; y < h; y++){
      for(var x = 0; x < w; x++){
        for(var inner_y = 0; inner_y < mult;inner_y++){
          for(var inner_x = 0; inner_x < mult;inner_x++){
            var ny = y * mult + inner_y;
            var nx = x * mult + inner_x;
            out[ny * n_w + nx] =  bicubic_value(arr_in,y,x,inner_y/mult,inner_x/mult, w, h);
            if(out[ny * n_w + nx] > max){
              max = out[ny * n_w + nx];
            } else if (out[ny * n_w + nx] < min) {
              min = out[ny * n_w + nx];
            }
          }
        }
      }
    }
    return {'data' : out, 'min':min,'max': max};
}

{% endhighlight %}

Here are a few samples of the results:

Hand no interpolation:
![Sample Image]({{ "/assets/images/thermal_camera/dsc02964_small.jpg" }})

Hand bilinear interpolation 4x:
![Sample Image]({{ "/assets/images/thermal_camera/dsc02960_small.jpg" }})

No interpolation, selfie:
![Sample Image]({{ "/assets/images/thermal_camera/no_interpolation_selfie.png" }})

Bilinear interpolation, selfie:
![Sample Image]({{ "/assets/images/thermal_camera/bilinear_selfie.png" }})

Bicubic interpolation, selfie:
![Sample Image]({{ "/assets/images/thermal_camera/bicubic_selfie.png" }})

I am very satisfied with the results I got from the bilinear interpolation. I tried out both algorithms in Java and tested with some images and I was not able to see a noticeable difference. For an aplication like this, I'd stick to the simpler bilinear interpolation, specially since there is not much room to get 4x4 temperature data.

There are a couple of applications I can think of for this sensor, as well as reasons to pair it with the ESP8266. Some of these could be: Anonymosly counting people, baby crib overhead temperature monitoring, stove monitoring. By using the ESP8266 in AP mode, it should be easy to put in place and configure the sensor by only using a phone.