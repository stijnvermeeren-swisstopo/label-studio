<img src="https://user-images.githubusercontent.com/12534576/192582340-4c9e4401-1fe6-4dbb-95bb-fdbba5493f61.png"/>

# Label Studio for Boreholes data extraction

For the official Label Studio documentation we refer to the parent repository [heartexlabs/label-studio](https://github.com/heartexlabs/label-studio).

# Deployment Guide
The below is a guide on how to deploy the label studio instance including their ml-backends. The deployment is meant for an EC2 (or other VM) and we assume that the infrastructure has already been set up.

The deployment steps are:
1. Prepare the data
2. Configure the environments for Docker Compose
3. Add the shared Docker network
4. Build and run `label-studio` and `label-studio-ml-backend` Docker containers
5. Setup your Label Studio instance

## 1. Data Preparation
The labeling project this deployment guide describes comes with fixed data sets that need to be prepared **before** the setup. The data preparation comes in two steps.
1. Create the png images for the pdf pages.
2. Upload png files as well as the pdf files to the EC2 instance.

### PNG file creation
Create a folder that contains all borehole profiles in pdf format that you'd like to annotate using label studio. Then use the script that comes with the `swissgeol-boreholes-dataextraction` project which is located [here](https://github.com/swisstopo/swissgeol-boreholes-dataextraction/blob/main/src/scripts/convert_pdf_to_png.py). To execute the script run `python convert_pdf_to_png.py --input-directory PATH_TO_PDF_FILES --output-directory PATH_TO_PNG_DIRECTORY`. This will create one file per pdf page for all PDFs in the input directory.

### File upload to EC2
Once you have created the png files, you have to upload both the pdf as well as the png directory to your EC2. We recommend using the `scp` command for this. The data can be placed anywhere where your user has access to. For example in the home directory of your remote user on the EC2. The data directory needs to have the following structure:
```
data/pdf/project1
data/png/project1
data/pdf/project2
data/png/project2
...
```
Your data directory contains two directories. One for all pdf files, and one for all png files. These two folders then contain a directory for each project.

Example upload command `$ scp -r ~/local_directory user@host.com:/home/ubuntu/data`.
Make sure to configure your ssh connection with the EC2 instance beforehand.

## 2. Configure Docker Compose environments

Mount the directory with the data to the `app` service in label studio as well as in all ml-backend services. You don't need to adjust the docker-compose directly, but you need to place a `.env` file in the same level as the respective `docker-compose.yml` files. That is, one `.env` file in the current label-studio directory, and one in [label-studio-ml-backend/label_studio_ml](https://github.com/redur/label-studio-ml-backend/tree/master/label_studio_ml). The content of the .env should be:
```.env
DATA_DIRECTORY_PATH=/absolute_path_to_your_data
```
There is also a file called `.env.example` which you can use.
The data directory needs to contain one or several subdirectories containing pdf files (the borehole profiles) and one additional subdirectory called `test_png` containing all png files for the borehole profiles (one png file per page).

## 3. Add a shared docker network
Run `sudo docker network create label-studio-network` on your EC2.

## 4. Build and run docker instances
We recommend to use the tmux shell extension to run the different docker containers. Tmux is pre-installed in EC2. Otherwise, see [github.com/tmux/tmux/wiki/Installing](https://github.com/tmux/tmux/wiki/Installing) for installation instructions.

Tmux helps you keep processes alive even if you kill your shell instance. A cheatsheet regarding tmux commands can be found [here](https://tmuxcheatsheet.com).

In order to build and run your docker instances on your EC2 instance, do:
1. Clone the repositories on your EC2.
2. Create a data directory and upload all data for your application in that repository. All png files need to be uploaded to a directory called `test_png` which is a direct child of your data directory.
3. Create a new tmux session called ml-backend and label-studio in your EC2: `tmux new -s ml-backend` &  `tmux new -s label-studio`.
4. Attach to the tmux session using `tmux a -t ml-backend` or `tmux a -t label-studio`. Navigate to the respective docker-compose and do `sudo docker compose build` to build all docker images.
5. Then, inside the respective tmux session do `sudo docker compose up` and your services should be up and running.

## 5. Setup label-studio
1. Navigate to the URL of your EC2 (make sure you allow inbound traffic to the port on which label-studio listens in the security group of your EC2.)
2. Create a new user, and log in using it.
   * If you don't see the option to sign up as a new user, you have to temporarily set the variable `LABEL_STUDIO_DISABLE_SIGNUP_WITHOUT_LINK` to `false` in [docker-compose.yml](docker-compose.yml) and restart the Docker container. After your initial login, you can revert the value back to `true`.
3. Create a new project from the UI by clicking on `Create Project`. Specify the name of the project (e.g., Data Extraction) and skip all the other setup steps and click on `Save`. 

![](assets/img/create_project.png)

4. Once you are on the main screen of your new project. Go to _Settings_ → _Cloud Storage_ → _Add Source Storage_.
   * Choose storage type `local files` and add the path to the png files of your project. This is a subdirectory of your previously mounted directory in your docker. Example: `/label-studio/files/png/project_name`
   * Make sure to enable the toggle `Treat every bucket object as a source file` otherwise the import will fail.
   * Once you are back on the `CLoud Storage` widget click on the button `Sync Storage`

![](assets/img/data_cloud.png)

5. Set up the labeling interface: Go to _Settings_ → _Labeling Interface_ → _Code_ and paste
```html
<View>
  <Image name="image" value="$ocr"/>

  <Labels name="label" toName="image">
    <Label value="Material Description" background="green"/>
    <Label value="Depth Interval" background="blue"/>
  <Label value="Coordinates" background="#FFA39E"/></Labels>

  <Rectangle name="bbox" toName="image" strokeWidth="3"/>
  <Polygon name="poly" toName="image" strokeWidth="3"/>

  <TextArea name="transcription" toName="image" editable="true" perRegion="true" required="true" maxSubmissions="1" rows="5" placeholder="Recognized Text" displayMode="region-list"/>
</View>
```
6. Add the ML Backends: Go to _Settings_ → _Model_ → _Connect Model_. Use the following settings (as shown in the screenshots below):
   * Name: _LayerExtractor_
     * Backend URL: http://stratigraphy-ml-backend:9090
   * Name: _TextExtractor_
     * Backend URL: http://text_extractor:9095
     * Interactive preannotations: _on_

<img src="readme-model-layer-extractor.png" width="400" />
<img src="readme-model-text-extractor.png" width="400" />

As an alternative to adding a model through the UI, you can do it using the below API call. In some versions of Label Studio, this is the only way of adding a second model. Check out the [API Reference](https://labelstud.io/api) for details about the API. You can generate your own token in the label-studio UI when clicking on your user in the top right corner.
```bash
curl -X POST -H 'Content-type: application/json' http://localhost:80/api/ml -H 'Authorization: Token TOKEN' --data '{"url": "http://BACKEND_HOST:BACKEND_PORT", "project": PROJECT_ID}'
```

Please note that here the variable `Project_ID` is the actual ID of the project. This is a number. Example: If this is the second project you create using label-studio then ID will be 2 and you wish to add the text extraction model to label-studio. The command would be the following: 

```bash
curl -X POST -H 'Content-type: application/json' http://localhost:80/api/ml -H 'Authorization: Token TOKEN' --data '{"url": "http://text_extractor:9095", "project": 2}'
```

If the models have a green dot next to their names, you know that the backends were detected and are up and running.

7. Now that you have set up the labeling you can invite new users. Click on the label-studio logo, then _Organization_ → _Add People_. This will provide you with a link where users can generate a user account and password.

## Infrastructure Settings
We use a t3.large instance with 32 GiB of disk storage.
- At least 8GiB of RAM.
- At least 32 GiB of disk storage (depending on the amount of borehole profiles you will need more).
- Security groups → make sure inbound rule for port 80 is set.

## Creating a second project
To create an additional annotation project on label studio you have to first upload the corresponding data to your ec2 and create a new directory for the borehole profiles. Place all png files in the existing test_png folder. Make sure there are no file name collisions. Then repeat step 5.

# Troubleshooting
- Can't specify label-studio version 
  - Check docker image; check version specified in remote repository.
- No space on device
  - Clear docker cache
- Permission denied for data
  - Change the permissions for the `mydata` folder.
- Can't add ML backend in the UI
  - Use label-studio API
- I can't access label studio from my browser.
  - Don't use IP but full EC2 name on AWS namespace.
- I don't have an API Token
  - Login to Label Studio.
  - Click on your user in the top right corner
  - _Account & Settings_
  - Copy the shown access token.
