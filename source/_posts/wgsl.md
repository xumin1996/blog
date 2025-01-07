---
title: wgsl
date: 2024-12-27 11:03:01
tags:
---

# 指针着色器 
## 介绍
![光环着色器](/wgsl/aura.png)
这个着色器只包含片元着色器代码，通过对像素点uv的计算显示指针

| 方法 | 描述 |
| --- | --- |
| atan2(y, x) | 计算y/x的反正切值,返回值的范围是−π,π。返回一个值使得tan(e)=y/x |
| smoothstep(min, max, x) | 平滑插值函数。公式：x * x * (3 - 2 * x) |
| step(edge, x) | 阶跃函数，x<edge返回0，x>edge返回1 |

## 代码
```wgsl
#import bevy_pbr::forward_io::{VertexOutput, FragmentOutput};
#import bevy_pbr::mesh_view_bindings::globals
#import bevy_render::view::View

/// Keep up-to-date with the rust definition!
struct AuraMaterial {
    unused: f32,
}

@group(0) @binding(0)   var<uniform> view: View;
@group(2) @binding(100) var<uniform> aura_mat: AuraMaterial;

// Colour picker tells us the values of the original..
// Darkish
// #CEAA4F
const GOLD = vec3f(0.807843, 0.666667, 0.309804);
const SPIKE_NUM: f32 = 9.0;
const SPIKE_LEN: f32 = 0.68;
const SPIKE_SPEED:f32 = 32.0;
const PI: f32 =  3.141592653589;

@fragment
fn fragment(in: VertexOutput) -> @location(0) vec4<f32> {
    var uv = in.uv;
    uv = uv * 2.0 - 1.0;
    let x =(atan2(uv.x, uv.y) / PI + 1) * SPIKE_NUM; // Divide the x coords by PI so they line up perfectly.

    // 计算光针边缘
    let f_x = fract(x);
    var m = min(f_x, 1.0 - f_x);
    m = m  + 0.5*length(uv);
    
    // 计算当前像素值:
    var c = smoothstep(0.5,  0.0, m);
    var col = vec3f(c);

    // 全局时间计算指针位置
    let time = globals.time;
    let time_circle_index = floor(time * SPIKE_SPEED) % (SPIKE_NUM * 2.0);
    let focus_length = min(abs(time_circle_index - x), abs(time_circle_index + 2*SPIKE_NUM - x));
    let is_focused_spike = step(0.5, focus_length);
    col *= mix(GOLD / 0.15, GOLD * 0.54, is_focused_spike);

    // 不显示中间
    let feet_mask = length(uv) - 0.25;
    col *= smoothstep(0.0, 0.09, feet_mask);

    // 输出
    var out = vec4f(col, 1.0);
    return out;
}
```

# 闪烁线条
## 介绍
在表面形成闪烁变化的网格,grid函数显示宽度为`1.0/GRID_RATIO`的线条
![](/wgsl/light.png)

## 代码
```wgsl
#import bevy_pbr::mesh_view_bindings::globals
#import bevy_sprite::mesh2d_vertex_output::VertexOutput

const GRID_RATIO:f32 = 40.;

@fragment
fn fragment(in: VertexOutput) -> @location(0) vec4<f32> {
    let t = globals.time;
    var uv = in.uv - 0.5;
    var col = vec3(0.0);

    uv *= 10.;
    let grid = grid(uv);
    let pal = palette(t / 2. );
    col = mix(col, pal, grid);
   
    return vec4<f32>(col, 1.0);
}

// 变幻彩色
fn palette(time : f32) -> vec3<f32> {
    let a = vec3<f32>(0.5, 0.5, 0.5);
    let b = vec3<f32>(0.5, 0.5, 0.5);
    let c = vec3<f32>(1.0, 1.0, 1.0);
    let d = vec3<f32>(0.263, 0.416, 0.557);

    return a + b * cos(6.28318 * (c * time + d));
}

// 显示网格
fn grid(uv: vec2<f32>)-> f32 {
    let i = step(fract(uv), vec2(1.0/GRID_RATIO));
    return max(i.x, i.y);
}

fn hsv2rgb(c: vec3<f32>) -> vec3<f32> {
    let K: vec4<f32> = vec4<f32>(1.0, 2.0 / 3.0, 1.0 / 3.0, 3.0);
    var p: vec3<f32> = abs(fract(vec3<f32>(c.x) + K.xyz) * 6.0 - K.www);
    return c.z * mix(K.xxx, clamp(p - K.xxx, vec3<f32>(0.0), vec3<f32>(1.0)), c.y);
}
```

# 随机粒子
物体表面随机生成随机运动的粒子.通过fft生成一个连续伪随机函数,随时间变化生成粒子的xy坐标,通过冰花过渡显示
```js
// js生成连续伪随机函数
s = ''
for (let i = 0;i<10;i++) {
  s += Math.random().toFixed(4) + '*sin('+Math.pow(2,i) +'*x)+' + Math.random().toFixed(4)  + '*cos('+Math.pow(2,i) +'*x)+'
}

```
![](/wgsl/points.png)

## 代码
```wgsl
#import bevy_pbr::mesh_view_bindings globals
#import bevy_sprite::mesh2d_vertex_output::VertexOutput

@fragment
fn fragment(in: VertexOutput) -> @location(0) vec4<f32> {
    let uv: vec2<f32> = in.uv;

    var m = 0.;
    let t: f32 = globals.time / 100;
    for (var i = 0; i < 30; i += 1) {
        let n:vec2<f32> = vec2(rand(t + f32(i )* 2.0), rand(t + 0.5 + f32(i) * 2.0));
        let d = length(uv - n);
        m += smoothstep(0.002, 0.001, d);
    }

    var col = vec3(m);
    return vec4(col, 1.0);
}

// fft构造的连续伪随机树
fn rand(x: f32) -> f32 {
    // fft 随机数
    var y = 0.1042*sin(1*x)+0.7563*cos(1*x)+0.4530*sin(2*x)+0.6678*cos(2*x)+0.8024*sin(4*x)+0.1780*cos(4*x)+0.2779*sin(8*x)+0.4869*cos(8*x)+0.2147*sin(16*x)+0.5170*cos(16*x)+0.0115*sin(32*x)+0.6059*cos(32*x)+0.9722*sin(64*x)+0.1842*cos(64*x)+0.9056*sin(128*x)+0.6755*cos(128*x)+0.1378*sin(256*x)+0.6769*cos(256*x)+0.2455*sin(512*x)+0.8363*cos(512*x);
    y = (y / 10) / 2 + 0.5;
    return min(max(0.0, y), 1.0);
}
```