# Conventions for multiscale images (pyramids)

Multiscale images, or scale pyramids, or image pyramids ([wikipedia](https://en.wikipedia.org/wiki/Pyramid_%28image_processing%29)), are a convenient representation of large image datasets. Typically, a pyramid with `N` levels is generated by downsampling an original image `N - 1` times by a constant factor per axis, e.g. 2. So creating a 3 level pyramid with isotropic downsampling from a 3D volume with a size of `[64, 64, 64]` pixels would yield 3 images:
```
input: 
-- Original image:      {size: [64, 64, 64], resolution: [1, 1, 1]}

output:
-- Original image:      {size: [64, 64, 64], resolution: [1, 1, 1]}
-- Downsampled image 1: {size: [32, 32, 32], resolution: [2, 2, 2]}
-- Downsampled image 2: {size: [16, 16, 16], resolution: [4, 4, 4]}
```
In full generality, each axis of the image may be downsampled by a different factor, and the downsampling factors may vary between levels. 

There are a variety of ways to handle the metadata associated with image pyramids. Downsampling is guaranteed to alter the resolution (i.e., the spacing between samples) of an image, and certain methods of image downsampling add a slight translation to the resulting images. Additionally, an image pyramid should have some metadata features that allow consuming software to detect that it is in fact a pyramid, and not simply a collection of images with monotonically shrinking sizes. 

When storing image pyramids in hierarchical array storage formats (namely, HDF5, N5, Zarr), there are a variety of extant approaches for representing the pyramid and handling metadata. 


## deprecated n5-viewer style ([source](https://github.com/saalfeldlab/n5-viewer/commit/4df02d4f9aadbfe4aa31fcded748fce57519a70c#diff-04c6e90faac2675aa89e2176d2eec7d8)) 
```
└─ root (optional) {"resolution" : [r_x, r_y, r_z]} 
        (optional) {"pixelResolution": {"dimensions": [r_x, r_y, r_z], "unit": 'm'},
        (optional) {"scales" : [[a, b ,c], [i*a, j*b, k*c], ... ]}
    ├── (required) s0 {} 
    ├── (required) s1 (optional, unless "scales" is not a group level attribute): {"downsamplingFactors": [a, b, c]})
    ...
```
Scale levels must be datasets in the same group with names `s{N}`, where `N` is the scale level starting at `s0` (full resolution). 

### modern n5-viewer style ([source](https://github.com/saalfeldlab/n5-viewer/commit/36e75fd88ebcbc88a64da9fb082a28f9b46ded21#diff-04c6e90faac2675aa89e2176d2eec7d8))
```
└─ root (optional) {"resolution" : [r_x, r_y, r_z]}  
        (optional) {"pixelResolution": {"dimensions": [r_x, r_y, r_z], "unit": 'm'}}
    ├── (required) s0 {} 
    ├── (required) s1 (required) {"downsamplingFactors": [a, b, c]}
    ...
```
* Like the deprecated `n5-viewer` style, scale levels must be datasets in the same group with names `s{N}`, where `N` is the scale level starting at `s0` (full resolution). 
* Both `n5-viewer` styles explicitly store the base resolution in one group-level attribute, and infer the resolution for different scale levels by combining that base resolution with the dataset-level `downsamplingFactors` attribute. This allows changing the resolution of all pyramid levels by adjusting a single attribute.
* The lack of `resolution` and `offset` attributes for the scale level datasets means they cannot be viewed in world coordinates outside the context of the entire pyramid group. 
* The offset for each scale level are not specified. This approach silently assumes a specific form of downsampling that applies an increasing coordinate offset to each scale level.

### New proposed COSEM style: 
```
└─ root (optional) {"gridSpacing": [r_x, r_y, r_z]} 
        (optional) {"origin": [o_x, o_y, o_z]} 
        (required) {"multiscale": [{"name": "foo",  [*foo.attributes]}, {"name" : "bar", [*bar.attributes]}, ...]}
    ├── foo (required) {"gridSpacing": [r_x, r_y, r_z], "origin": [o_x, o_y, o_z]} 
    ├── bar (required) {"gridSpacing": [a * r_x, b * r_y, c * r_z], "origin": origin_func([o_x, o_y, o_z])}
    ...
(origin_func depends on the type of downsampling used)
(a, b, and c are numeric real numbers >= 1.0)
(personally I like "multiscale" more than "pyramid")
```
* This scheme avoids specifying scale levels with pre-determined dataset names. Instead, the pyramid is specified in the group attributes as a mapping between dataset names and dataset attributes. This allows mixing pyramid and non-pyramid datasets in the same group, or potentially specifying pyramids that span multiple groups. It is not clear which dataset attributes should be stored in this group attribute -- maybe only the parameters that change with downsampling (i.e., `resolution` + `offset`)?
* The attributes of scale level datasets are generic self-describing dataset attributes like `gridSpacing` and `origin`, which means these datasets can be viewed in real space without the group-level attributes. 
* Root attributes include `gridSpacing` and `origin`, which can be used to transform the dataset attributes with the same name. This allows rapidly rescaling the resolution of all scale levels, similar to the functionality of combination of group-level resolution attributes with the dataset-level `downsamplingFactors` attribute in the n5-viewer styles. Pyramids could also be defined this way using a transformation matrix instead of separate `gridSpacing` and `origin` attributes for each dataset. 
* A major downside to this scheme is that the `multiscale` group-level attribute is relatively complex, and cannot be natively represented as an `HDF5` attribute; instead, such an attribute must be written & read as a string and then parsed as JSON by consuming programs. 

### Contemporary new proposed COSEM style (i.e. elaborated OME-ZARR spec)
After a [discussion on github](https://github.com/zarr-developers/zarr-specs/issues/50) about group-level multiresolution metadata, we at COSEM have settled on using the OME-Zarr spec, which in our implementation looks like this:

*Group metadata*:
```json
{
"multiscales": [
        {
            "datasets": [
                {
                    "path": "s0",
                    "transform": { # Implementation detail: these
                        "axes": [  # axis values are all listed in C order. 
                            "z",
                            "y",
                            "x"
                        ],
                        "scale": [
                            5.24,
                            4.0,
                            4.0
                        ],
                        "translate": [
                            0.0,
                            0.0,
                            0.0
                        ],
                        "units": [
                            "nm",
                            "nm",
                            "nm"
                        ]
                    }
                },
                {
                    "path": "s1",
                    "transform": {
                        "axes": [
                            "z",
                            "y",
                            "x"
                        ],
                        "scale": [
                            10.48,
                            8.0,
                            8.0
                        ],
                        "translate": [
                            2.62,
                            2.0,
                            2.0
                        ],
                        "units": [
                            "nm",
                            "nm",
                            "nm"
                        ]
                    }
                }
             ]
         }
    ]
}
```
*Dataset metadata:*
```json
{
    "transform": {
        "axes": [
            "z",
            "y",
            "x"
        ],
        "scale": [
            5.24,
            4.0,
            4.0
        ],
        "translate": [
            0.0,
            0.0,
            0.0
        ],
        "units": [
            "nm",
            "nm",
            "nm"
        ]
    }
}
```

Notbale here is  the `transform` property, which appears in the dataset metadata and group metadata. This object conveys the following information:
- The names and units of each array axis (in C order). 
- How the array axes of the source dataset should be mapped into physical space through translation and scaling.

This `transform` object does not support coordinate transformations like rotation and shear.

N.B.: In practice, for COSEM datasets there may be other (redundant) elements of spatial metadata present in group and dataset attributes; The content of this metadata is guranteed to agree with the aforedescribed `transform` metadata EXCEPT in the axis ordering, which may be F ordered (e.g. for metadata that will be consumed by Java-based tooling, or neuroglancer). The ambiguity here is unfortunate, and should be resolved with an unambiguous specification of axis ordering.