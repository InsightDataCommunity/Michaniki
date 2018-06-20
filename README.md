# Michaniki
Michaniki is a web platform that allows user to create a image classifier for custom images, based on pre-trained CNN models. It also allows to use the customized CNN to pre-label images, and re-train the customized CNN when more training data become avaliable.n

Michaniki is currently serveing as RESTful APIs without front-end UI.

---
## Development
Everything is wrapped in Docker, here are the core packages used in *Michaniki*.
* Python 2.7.3
* uWSGI
* Flask
* Docker (built on 17.12.0-ce)
* Docker Compose (1.18.0)
* Keras + Tensorflow

## Setup environment
#### 1. Install Docker:
On Linux Machines (Ubuntu/Debian) -- UNTESTED!

```bash
sudo apt-cache policy docker-ce
sudo apt-get install docker-ce=17.12.0-ce
```

On Mac :
Download [Docker from here](https://store.docker.com/editions/community/docker-ce-desktop-mac) and install it.

#### 2. Install Docker Compose
Docker compose can start multiple containers at the same time.
On Linux Machines (Ubuntu/Debian)  -- UNTESTED!:

```bash
sudo curl -L https://github.com/docker/compose/releases/download/1.18.0/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

On Mac:
The package of Docker-CE for Mac already come with docker compose.

#### Install Git:
[Install git](https://git-scm.com/downloads) for the appropriate system, if you don't have yet.

## Build and Start *Michaniki*
#### Clone the repo:

```bash
git clone https://github.com/InsightDataCommunity/Michaniki
```

#### Export Your AWS Credentials to Host Environment.
*Michaniki* needs to access S3 for customized images. You should create an IAM user at [AWS](https://aws.amazon.com/), if you don't yet have an AWS account, create one.

Then, export credentials to your local environment:
```bash
export AWS_ACCESS_KEY_ID=YOUR_ACCESS_ID
export AWS_SECRET_ACCESS_KEY=YOUR_ACCESS_KEY
```

If success, you should be able to see your credentials by:

```bash
echo $AWS_ACCESS_KEY_ID
echo $AWS_SECRET_ACCESS_KEY
```

Docker will access these environment variables and load them to docker.

#### Start the Docker Containers:
move to the directory where you cloned *Michaniki* , and run:
```bash
docker-compose up --build
```

If everything goes well, you should start seeing the building message of the docker containers:
```
Building michaniki_client
Step 1/9 : FROM continuumio/miniconda:4.4.10
 ---> 531588d20a85
Step 2/9 : RUN apt-get update && apt-get install -y --no-install-recommends apt-utils
 ---> Using cache
 ---> a356e16a75e7
...
```

The first time of building might take few minutes, once finished, you should see the *uWSGI* server, *inference* server running, something like this:
```
...
michaniki_client_1  | *** uWSGI is running in multiple interpreter mode ***
michaniki_client_1  | spawned uWSGI master process (pid: 8)
michaniki_client_1  | spawned uWSGI worker 1 (pid: 18, cores: 2)
michaniki_client_1  | spawned uWSGI worker 2 (pid: 19, cores: 2)
michaniki_client_1  | spawned uWSGI worker 3 (pid: 21, cores: 2)
michaniki_client_1  | spawned uWSGI worker 4 (pid: 24, cores: 2)
```

Then *michaniki* is ready for you.

## Use *Michaniki*:
*Michaniki* currently provides 3 major APIs. To test *Michaniki*, I recommend to test the APIs using [POSTMAN](https://www.getpostman.com/), while the examples below are in terminal, using `cURL`

#### 0. Welcome Page
If *Michaniki* is running correctly, go to `http://127.0.0.1:3031/` in your web browser, you should see the welcome message of *Michaniki*.

#### 1. Predict a Image with InceptionV3
*Michaniki* can classify any image to ImageNet labels using a pre-trained InceptionV3 CNN:

```bash
curl -X POST \
  http://127.0.0.1:3031/inceptionV3/predict \
  -H 'Cache-Control: no-cache' \
  -H 'Postman-Token: eeedb319-2218-44b9-86eb-63a3a1f62e14' \
  -H 'content-type: multipart/form-data; boundary=----WebKitFormBoundary7MA4YWxkTrZu0gW' \
  -F image=@*PATH/TO/YOUR/IMAGE* \
  -F model_name=base
```

replace the name after ``` image=@ ``` by the path of the image you want to run. *Michaniki* will return the image labels and probabilities, ranked from high to low, in `json` format:

The model name *base* is used to refer to the basic InceptionV3 model.

```
{
    "data": {
        "prediction": [
            {
                "label": "Egyptian cat",
                "probability": 0.17585527896881104
            },
            {
                "label": "doormat, welcome mat",
                "probability": 0.057334817945957184
            },
			...
```

#### 2. Create a New Classifier Using Custom Image Dataset:
*Michaniki* can do transfer learning on the pre-trained InceptionV3 CNN (without the top layer), and create a new CNN for users' image dataset.

**The new image dataset should be stored at S3 first, with the directory architecture in S3 should look like this**:
```
.
+-- YOUR_BUCKET_NAME
|   +-- YOUR_MODEL_NAME
|   	+-- train
|			+-- class1
|			+-- class2
|			...
|		+-- val
|			+-- class1
|			+-- class2
|			...
```
The folder name you give to *YOUR_MODEL_NAME* will be used to identify this model once it get trained.

To call this API, do:
```bash
curl -X POST \
  http://127.0.0.1:3031/inceptionV3/transfer \
  -H 'Cache-Control: no-cache' \
  -H 'Postman-Token: 4e90e1d6-de18-4501-a82c-f8a878616b12' \
  -H 'content-type: multipart/form-data; boundary=----WebKitFormBoundary7MA4YWxkTrZu0gW' \
  -F train_bucket_name=YOUR_BUCKET_NAME \
  -F train_bucket_prefix=models/YOUR_MODEL_NAME
```
*Michaniki* will use the provided images to create a new model to classify the classes you provided in the S3 folder. Once the transfer learning is done, you can use the new model to label images by pass **YOUR_MODEL_NAME** to the inference API desribed eariler.

#### 3. Resume training on existing model:

