# How to import images from online sources into napari
In this tutorial we'll directly import images from publicly available online datasets into napari. 

It's not always that we'd have all the data on our system. Maybe we need to view some data available on online platforms or maybe we are working as a team on github and we need to view some images that are not already downloaded on our system. 

This tutorial helps us import those images directly into napari and cuts down a lot of unnecessary hassle. We have given code along with explanations and some useful links in case somebody wants to explore more.

For the purpose of example we have chose this publicly available [online dataset](https://idr.openmicroscopy.org/ "IDR from Open Microscopy"). Its the Image Data resource from Open Microscopy.

Step 1 : Open this link and find the dataset you want to view: [https://idr.openmicroscopy.org/](https://idr.openmicroscopy.org/)
![Homepage](https://drive.google.com/file/d/1Y75_78QItfhWOGkxiOktWd7F7UZ2TSiy/view?usp=sharing)

Step 2: Click on any dataset and you’ll be shifted to a viewport like below: 
![Thumbnails](https://drive.google.com/file/d/18Mh-IkpkpdaQfJrfvIT6diXv01MzGLMz/view?usp=sharing)

Step 3: Click on any image that you would want to view in napari 
![Final Image]("https://drive.google.com/file/d/1BXnLnMbW85iaiCw9E0LQcFzngiqeSz97/view?usp=sharing")

Step 4: Note down the image ID
Step 5: Open Command Line and type in "napari"
Step 6: Type or Copy-Paste the following lines of code into the IPython console

We'll be using requests library from python hence first we create HTTP Session.
```
import requests
import shutil 
import napari 
IDR_BASE_URL = "https://idr.openmicroscopy.org"
INDEX_PAGE = "%s/webclient/?experimenter=-1" % IDR_BASE_URL
# create http session using requests library. 
with requests.Session() as session:
    request = requests.Request('GET', INDEX_PAGE)
    prepped = session.prepare_request(request)
    response = session.send(prepped)
    if response.status_code != 200:
        response.raise_for_status()
```

Now we create our IDR image link into a useable URL and download it onto our system.
The image ID used below is an example. You can replace it with the image ID of your chosen image.
Also you can change the name of image file.
```python
IMAGE_DETAILS_URL = "{base}/webclient/imgData/{image_id}/"
IMAGE_ID = 	9627732
# Image ID can be changed - It'll be unique to each image of the dataset
qs = {'base': IDR_BASE_URL, 'image_id': IMAGE_ID}
url = IMAGE_DETAILS_URL.format(**qs)
RENDER_IMAGE = "{base}/webgateway/render_image/{image_id}/0/0/"
image_url = RENDER_IMAGE.format(**qs)
resp = requests.get(image_url, stream=True)
local_file = open('local_image.jpg', 'wb')
resp.raw.decode_content = True
shutil.copyfileobj(resp.raw, local_file)
del resp
```
[Explore More about IDR here](https://github.com/IDR/idr-notebooks/blob/master/IDR_API_example_script.ipynb "IDR code examples")
[Explore More about importing URL to python Here](https://www.dev2qa.com/how-to-download-image-file-from-url-use-python-requests-or-wget-module/ "dev2qa")


Finally we load that image into napari. 
You must change the name of the path of your image. 
```
import napari

napari.view_path('/Users/Ira/Desktop/local_image.jpg')
# We don't need to use "with napari.gui_qt():" because we are already in the Ipython console
```
Voila! A new napari viewer opens with your required image and all you needed was the image URL. 
Now you can proceed on analyzing and layering and playing around with your imported image. 

Thankyou. 
