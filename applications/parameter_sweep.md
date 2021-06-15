# Lazy parameter sweep in napari using Dask and magicgui

A parametric sweep allows for a parameter to be continuously varied through a range of user-defined values.
By using `napari` to perform a parameter sweep, the user may call a method with different parameter values (by moving a slider in napari), while observing the effect in the viewer.

This tutorial is designed to help users visualize parameter sweeps using napari, [Dask](https://dask.org/) and [magicgui](https://magicgui.readthedocs.io/). 
We start with the example of _lazy parameter sweep_ for a 2D parametric function and then proceed to real-world examples.

The python library, Dask, allows us to perform our tasks [_lazily_](https://en.wikipedia.org/wiki/Lazy_evaluation). 
This means that rather than immediately performing the tasks that a function has to perform, it records the tasks and computes them only when required.

## 1. Lazy parameter sweep using napari and Dask

We begin with an example demonstrating parametrically evaluating functions on arrays.
First we import the following libraries:
```python
import numpy as np
from scipy.ndimage.interpolation import shift
from dask import delayed
import dask.array as da
import napari
from itertools import product
```

We then define a 2D periodic function by generating 200 values between -5π and +5π, and take their sine and cosine, as illustrated below:
```python
# define a 2D periodic pattern
x = np.linspace(-np.pi, np.pi, 200) * 5
y = x
field = np.outer(np.sin(y) , np.cos(x))
```

We now define a wrapper function that delays (i.e. makes "lazy") the execution of [`scipy.ndimage.interpolation.shift`](https://docs.scipy.org/doc/scipy/reference/generated/scipy.ndimage.shift.html) which shifts an array. 
This is done using the [`dask.delayed`](https://docs.dask.org/en/latest/delayed.html) function. 
Rather than executing your function immediately, it will defer execution, placing the function and its arguments into a task graph.<sup>[1]</sup>
```python
def dashift(array, shifts):
    delshift = delayed(shift)
    return da.from_delayed(delshift(array, shifts), dtype='float64', shape=array.shape)
```

Next, we define the desired range of x-shifts and y-shifts:
```python  
x_shifts = np.arange(-10, 10)
y_shifts = x_shifts
n_s = len(x_shifts)
```

In the code snippet below, we take the product of arrays `y_shifts` and `x_shifts` to get tuples with all possibilities of `(y,x)`. 
These tuples are passed to `dashift`, our delayed shift function, and the output arrays are stacked using [`dask.stack`](https://docs.dask.org/en/latest/array-stack.html), to create an array of arrays. 
Finally, we use `reshape` and `rechunk` to get a 4D array with the required dimensions, and we build the lazily shifted array:
```python
shifted = (
    da.stack(dashift(field, arg) for arg in product(y_shifts, x_shifts))
    .reshape((n_s, n_s, *field.shape))
    .rechunk((1, 1, -1, -1))
    )
```

Following is a sample [task graph visualization](https://docs.dask.org/en/latest/graphviz.html) for smaller `x_shifts` and `y_shifts` arrays:

![image]({{ '/assets/tutorials/dask_taskgraph.png' | relative_url }})

We now use napari to visualize this parametric function:
```python
with napari.gui_qt():
    viewer = napari.view_image(shifted - field)
    viewer.dims.axis_labels = ["Y shift", "X shift", "Y", "X"]
```
![image]({{ '/assets/tutorials/lazy_param_sweep.gif' | relative_url }})


## 2. Lazy parameter sweep with napari and Dask to find the best thresholding technique

Now, we proceed to a real-world application of the above concept. 
Making use of lazy parameter sweep in napari can simplify many tasks, such as manually finding the best thresholding technique, or deciding the desired block size or window size to be passed as parameters for thresholding.

In the example below, we use [`scikit-image`](https://scikit-image.org/) to import the image of a scanned page of text. 
Then we apply three different thresholding techniques, from the [`skimage.filters`](https://scikit-image.org/docs/dev/api/skimage.filters.html) library, with a slider to control the threshold block or window size:
```python
# importing required libraries
from skimage import data
import numpy as np
import napari
import skimage.filters
import dask.array as da
from dask import delayed

# importing image of scanned page of text
img = data.page()

# creating a list of threshold types
thresholds = ["threshold_local", "threshold_sauvola", "threshold_niblack"]
```

Next, we define a wrapper function, `del_thresh` that delays the execution of the thresholding function, passed as parameter, by making it lazy. 
`apply_thresh` is the lazy version of the thresholding function passed to `del_thresh`. 
We use [`da.from_delayed`](https://docs.dask.org/en/latest/array-creation.html) to create a single-chunked Dask array from the Dask delayed array by providing `dtype` and `shape`:
```python
def del_thresh(img, threshold_type, size):
    apply_thresh = delayed(getattr(skimage.filters, threshold_type))
    return da.from_delayed(apply_thresh(img, size), dtype=img.dtype, shape=img.shape)
```

We then use our `del_thresh` function to threshold the image by applying all the threshold techniques in `thresholds` list, for all odd sizes from 1 through 99. 
The resultant image layers are then appended to the `images` array.
```python
images = []
for threshold_type in thresholds:
    stacked = da.stack(img > del_thresh(img, threshold_type, size) for size in np.arange(1, 100, 2))
    images.append(stacked)
```

Finally, we visualize the thresholded images with napari. 
We split the stacked `images` array into constituent image layers for different threshold types using `channel_axis=0` as follows:
```python
with napari.gui_qt():
    viewer = napari.view_image(img, name="page")
    viewer.add_image(da.stack(images), channel_axis=0, name=thresholds, colormap="gray")
```

![image]({{ '/assets/tutorials/lazy_thresholding.gif' | relative_url }})

## 3. Parameter Sweep with napari using the magicgui widget

To auto-generate a user interface that lets us explore different thresholding techniques, we can use [magicgui](https://magicgui.readthedocs.io/en/latest/). 
Here, the thresholded image is computed only for the layer and thresholding technique selected from the drop-down menus; and the value of `block_size` or `window_size` for the current threshold method is controlled by a slider.

First, we import the following libraries:
```python
from magicgui import magicgui
from magicgui._qt import QDoubleSlider
from napari import Viewer, gui_qt, layers
from skimage import data, filters
from math import floor
```

We then create a viewer with napari, and add the `data.page()` image layer to try out parameter sweep with different thresholding techniques. 
We use the `magicgui` decorator for the method `apply_threshold` to turn it into a magicgui.

The return annotation `-> layers.Image` for the `apply_threshold` function tells `magicgui` to add an image layer to the viewer to display the result of thresholding.

`auto_call = True` tells magicgui to call the `apply_threshold` function whenever the user changes one of the parameters in the GUI. 
The `QDoubleSlider` is used to ensure that the size parameter slider is interpreted as a `float` type, and we offer the `choices` to threshold an image layer using the adaptive, Sauvola and Niblack techniques:
```python
with gui_qt():
    # creating a viewer and adding the data.page() image layer
    viewer = Viewer()
    viewer.add_image(data.page(), name="page")

    # passing size as (2* floor(size/2)+1), as block-size and window size
    # have to be odd natural numbers
    @magicgui(
        auto_call=True,
        size={"widget_type": QDoubleSlider, "maximum": 99, "fixedWidth": 400},
        technique={"choices": ["local", "sauvola", "niblack"]},
    )
    def apply_threshold(layer: layers.Image, size=50, technique="local") -> layers.Image:
        if layer:
            func = getattr(filters, "threshold_" + technique)
            return layer.data > func(layer.data, (2 * floor(size / 2) + 1))
```

Then, we instantiate our magicgui widget and connect it to the current viewer for updation of layer drop-down choices. 
We also add the GUI as a widget to our viewer:
```python
    gui = apply_threshold.Gui()
    gui.parentChanged.connect(gui.refresh_choices)
    viewer.layers.events.changed.connect(lambda x: gui.refresh_choices("layer"))
    viewer.window.add_dock_widget(gui)
```

![image]({{ '/assets/tutorials/magicgui.gif' | relative_url }})

---

### Further Reading:
- [Using Dask and napari to process & view large datasets](https://napari.org/tutorials/applications/dask)

### References:
- [Documentation on dask.delayed](https://docs.dask.org/en/latest/delayed.html)
- [Introducing napari: a fast n-dimensional image viewer in Python](https://ilovesymposia.com/2019/10/24/introducing-napari-a-fast-n-dimensional-image-viewer-in-python)
- [magicgui docs for napari parameter sweeps](https://magicgui.readthedocs.io/en/latest/examples/napari_parameter_sweep)
