<img src="https://user-images.githubusercontent.com/12534576/192582340-4c9e4401-1fe6-4dbb-95bb-fdbba5493f61.png"/>

# Label Studio for Boreholes data extraction

For the official Label Studio documentation we refer to the parent repository [heartexlabs/label-studio](https://github.com/heartexlabs/label-studio).

# Deployment Guide
The below is a guide on how to deploy the label studio instance including their ml-backends. The deployment is meant for an EC2 (or other VM) and we assume that the infrastructure has already been set up.

The deployment steps are:
1. Configure the environments for Docker Compose
2. Add the shared Docker network
3. Build and run `label-studio` and `label-studio-ml-backend` Docker containers
4. Setup your Label Studio instance

## 1. Configure Docker Compose environments

Mount the directory with the data to the `app` service in label studio as well as in all ml-backend services. You don't need to adjust the docker-compose directly, but you need to place a `.env` file in the same level as the respective `docker-compose.yml` files. That is, one .env file in the current label-studio directory, and one in [label-studio-ml-backend/label_studio_ml](https://github.com/redur/label-studio-ml-backend/tree/master/label_studio_ml). The content of the .env should be:
```.env
DATA_DIRECTORY_PATH=/absolute_path_to_your_data
```
The data directory needs to contain one or several subdirectories containing pdf files (the borehole profiles) and one additional subdirectory called `test_png` containing all png files for the borehole profiles (one png file per page).

## 2. Add a shared docker network
Run `sudo docker network create label-studio-network` on your EC2.

## 3. Build and run docker instances
We recommend to use the tmux shell extension to run the different docker containers. Tmux is pre-installed in EC2. Otherwise, see [github.com/tmux/tmux/wiki/Installing](https://github.com/tmux/tmux/wiki/Installing) for installation instructions.

Tmux helps you keep processes alive even if you kill your shell instance. A cheatsheet regarding tmux commands can be found [here](https://tmuxcheatsheet.com).

In order to build and run your docker instances on your EC2 instance, do:
1. Clone the repositories on your EC2.
2. Create a data directory and upload all data for your application in that repository. All png files need to be uploaded to a directory called `test_png` which is a direct child of your data directory.
3. Create a new tmux session called ml-backend and label-studio in your EC2: `tmux new -s ml-backend` &  `tmux new -s label-studio`.
4. Attach to the tmux session using `tmux a -t ml-backend` or `tmux a -t label-studio`. Navigate to the respective docker-compose and do `sudo docker compose build` to build all docker images.
5. Then, inside the respective tmux session do `sudo docker compose up` and your services should be up and running.


## 4. Setup label-studio
1. Navigate to the URL of your EC2 (make sure you allow inbound traffic to the port on which label-studio listens in the security group of your EC2.)
2. Create a new user, and log in using it.
   * If you don't see the option to sign up as a new user, you have to temporarily set the variable `LABEL_STUDIO_DISABLE_SIGNUP_WITHOUT_LINK` to `false` in [docker-compose.yml](docker-compose.yml) and restart the Docker container. After your initial login, you can revert the value back to `true`.
3. Create a new project from the UI.
4. Go to _Settings_ → _Cloud Storage_ → _Add Source Storage_.
   * Choose storage type `local files` and add the path to the data (a subdirectory of the previously mounted ``)
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
6. Add the ML Backends: Go to _Settings_ → _Model_ → _Add Model_. Use the following settings (as shown in the screenshots below):
   * Name: _LayerExtractor_
     * Backend URL: http://stratigraphy-ml-backend:9090
   * Name: _Text Extractor_
     * Backend URL: http://text_extractor:9095
     * Interactive preannotations: _on_

<img src="image-2.png" width="400" />
<img src="image-3.png" width="400" />

As an alternative to adding a model through the UI, you can do it using the below API call. In some versions of Label Studio, this is the only way of adding a second model. Check out the [API Reference](https://labelstud.io/api) for details about the API. You can generate your own token in the label-studio UI when clicking on your user in the top right corner.
```bash
curl -X POST -H 'Content-type: application/json' http://localhost:80/api/ml -H 'Authorization: Token TOKEN' --data '{"url": "http://BACKEND_HOST:BACKEND_PORT", "project": PROJECT_ID}'
```

If the models have a green dot next to their names, you know that the backends were detected and are up and running.

7. Now that you have set up the labeling you can invite new users. Click on the label-studio logo, then _Organization_ → _Add People_. This will provide you with a link where users can generate a user account and password.

## Infrastructure Settings
We use a t3.large instance with 32 GiB of disk storage.
- At least 8GiB of RAM.
- At least 32 GiB of disk storage (depending on the amount of borehole profiles you will need more).
- Security groups → make sure inbound rule for port 80 is set.


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
