# GCP DevOps: Building a CI/CD Pipeline

This guide provides a step-by-step tutorial for setting up a basic Continuous Integration (CI) pipeline on Google Cloud Platform. You will learn how to:

1.  Create a private GitHub repository.
2.  Develop a simple Python Flask application.
3.  Containerize the application using Docker.
4.  Build and store the Docker image using Cloud Build and Artifact Registry.
5.  Automate builds using a Cloud Build trigger connected to your GitHub repository.
6.  Deploy and test your containerized application on a Compute Engine VM.

## Table of Contents

  - [Task 1: Create a Git Repository](https://www.google.com/search?q=%23task-1-create-a-git-repository)
  - [Task 2: Create a Simple Python Application](https://www.google.com/search?q=%23task-2-create-a-simple-python-application)
  - [Task 3: Define a Docker Build](https://www.google.com/search?q=%23task-3-define-a-docker-build)
  - [Task 4: Manage Docker Images with Cloud Build and Artifact Registry](https://www.google.com/search?q=%23task-4-manage-docker-images-with-cloud-build-and-artifact-registry)
  - [Task 5: Automate Builds with Triggers](https://www.google.com/search?q=%23task-5-automate-builds-with-triggers)
  - [Task 6: Test Your Build Changes](https://www.google.com/search?q=%23task-6-test-your-build-changes)

-----

## Task 1: Create a Git Repository

First, you will create a private Git repository on GitHub. This repository will store your source code and trigger a CI pipeline when new code is pushed.

1.  In the Google Cloud Console, click **Activate Cloud Shell** (the terminal icon in the top right). If prompted, click **Continue**.

2.  Run the following command in Cloud Shell to install the GitHub CLI:

    ```sh
    curl -sS https://webi.sh/gh | sh
    ```

3.  Log in to the GitHub CLI. Press `Enter` to accept the default options and follow the instructions to authenticate via the browser.

    ```sh
    gh auth login
    ```

4.  Confirm you are logged in by retrieving your GitHub username:

    ```sh
    gh api user -q ".login"
    ```

5.  Create an environment variable to store your GitHub username:

    ```sh
    GITHUB_USERNAME=$(gh api user -q ".login")
    ```

6.  Confirm the variable was created correctly:

    ```sh
    echo ${GITHUB_USERNAME}
    ```

7.  Set your global git credentials. **Replace `<YOUR_EMAIL>` with your actual email address.**

    ```sh
    git config --global user.name "${GITHUB_USERNAME}"
    git config --global user.email "<YOUR_EMAIL>"
    ```

8.  Create a new, empty, private GitHub repository named `devops-repo`:

    ```sh
    gh repo create devops-repo --private
    ```

9.  Create a local directory for your project:

    ```sh
    mkdir gcp-course
    cd gcp-course
    ```

10. Clone the empty repository to your Cloud Shell instance. If prompted, click **Authorize**.

    ```sh
    gh repo clone devops-repo
    ```

    > **Note:** You may see a warning that you have cloned an empty repository. This is expected.

11. Change to the new repository directory:

    ```sh
    cd devops-repo
    ```

## Task 2: Create a Simple Python Application

Next, create a simple Python Flask web application. This will be the source code for your CI pipeline.

1.  In Cloud Shell, click **Open Editor**.

2.  In the editor's explorer tree, navigate to the `gcp-course/devops-repo` folder.

3.  Create a new file named `main.py` inside the `devops-repo` folder and add the following code:

    ```python
    # main.py
    from flask import Flask, render_template, request

    app = Flask(__name__)

    @app.route("/")
    def main():
        model = {"title": "Hello DevOps Fans."}
        return render_template('index.html', model=model)

    if __name__ == "__main__":
        app.run(host='0.0.0.0', port=8080, debug=True, threaded=True)
    ```

4.  Create a new folder named `templates` inside the `devops-repo` folder.

5.  Inside the `templates` folder, create a new file named `layout.html` with the following content:

    ```html
    <!doctype html>
    <html lang="en">
    <head>
        <title>{{model.title}}</title>
        <link rel="stylesheet" href="https://stackpath.bootstrapcdn.com/bootstrap/4.4.1/css/bootstrap.min.css">
    </head>
    <body>
        <div class="container">
            {% block content %}{% endblock %}
            <footer></footer>
        </div>
    </body>
    </html>
    ```

6.  Inside the `templates` folder, create another new file named `index.html`:

    ```html
    {% extends "layout.html" %}
    {% block content %}
    <div class="jumbotron">
        <div class="container">
            <h1>{{model.title}}</h1>
        </div>
    </div>
    {% endblock %}
    ```

7.  In the root of the `devops-repo` folder, create a new file named `requirements.txt` to manage dependencies:

    ```txt
    # requirements.txt
    Flask>=2.0.3
    ```

8.  Save all your changes by pressing `CTRL + S` in the editor.

9.  Now, commit your application code to the repository. In the Cloud Shell terminal, run the following commands:

    ```sh
    # Ensure you are in the correct directory
    cd ~/gcp-course/devops-repo

    # Add all new files to be tracked by Git
    git add --all

    # Commit the changes locally
    git commit -a -m "Initial Commit"

    # Push the commit to your remote GitHub repository
    git push origin main
    ```

## Task 3: Define a Docker Build

Create a `Dockerfile` to define how your application's container image will be built.

1.  In the Cloud Shell Editor, ensure the `devops-repo` folder is selected.

2.  Create a new file in the root of the folder named `Dockerfile` (no extension).

3.  Add the following content to the `Dockerfile`. This configuration uses a Python base image, copies your application code, installs dependencies, and specifies the command to run the application using `gunicorn`.

    ```dockerfile
    # Dockerfile

    # Use an official Python runtime as a parent image
    FROM python:3.9

    # Set the working directory in the container
    WORKDIR /app

    # Copy the current directory contents into the container at /app
    COPY . .

    # Install production-ready web server and app dependencies
    RUN pip install gunicorn
    RUN pip install -r requirements.txt

    # Set the port the container will listen on
    ENV PORT=80

    # Command to run the application
    CMD exec gunicorn --bind :$PORT --workers 1 --threads 8 main:app
    ```

4.  Save the `Dockerfile` (`CTRL + S`).

## Task 4: Manage Docker Images with Cloud Build and Artifact Registry

Use Cloud Build to build your Docker image and store it securely in Artifact Registry.

1.  Return to the Cloud Shell terminal and ensure you are in the correct directory:

    ```sh
    cd ~/gcp-course/devops-repo
    ```

2.  The `DEVSHELL_PROJECT_ID` environment variable contains your project ID. View it to confirm:

    ```sh
    echo $DEVSHELL_PROJECT_ID
    ```

3.  Create a new Docker repository in Artifact Registry:

    ```sh
    gcloud artifacts repositories create devops-repo \
        --repository-format=docker \
        --location=europe-west4
    ```

4.  Configure Docker to authenticate with your new Artifact Registry repository:

    ```sh
    gcloud auth configure-docker europe-west4-docker.pkg.dev
    ```

5.  Submit your first build to Cloud Build. This command builds the image using your `Dockerfile` and tags it with `v0.1`.

    ```sh
    gcloud builds submit --tag europe-west4-docker.pkg.dev/$DEVSHELL_PROJECT_ID/devops-repo/devops-image:v0.1 .
    ```

6.  To verify, navigate to **Artifact Registry** in the Google Cloud Console. You should see your new `devops-image` in the `devops-repo` repository.

7.  Now, test the image by deploying it to a VM. In the Google Cloud Console, navigate to **Compute Engine \> VM Instances**.

8.  Click **Create Instance** and configure it as follows:

      - Under **Container**, click **DEPLOY CONTAINER**.
      - **Container image**: Enter the full path to your image: `europe-west4-docker.pkg.dev/qwiklabs-gcp-02-cae7e00e7a13/devops-repo/devops-image:v0.1` and click **SELECT**.
      - Under **Firewall**, check **Allow HTTP traffic**.
      - Leave all other settings as default and click **Create**.

9.  Once the VM is running, click its external IP address. A new browser tab should open and display "Hello DevOps Fans."

10. Finally, save the new `Dockerfile` to your Git repository:

    ```sh
    cd ~/gcp-course/devops-repo
    git add Dockerfile
    git commit -m "Added Docker Support"
    git push origin main
    ```

## Task 5: Automate Builds with Triggers

Create a Cloud Build trigger to automatically build your image every time you push a change to your GitHub repository.

1.  In the Cloud Console, navigate to **Cloud Build** and go to the **Settings** page. Enable the **Cloud Build** service API if it is not already.

2.  Go to the **Triggers** page and click **Create trigger**.

3.  Configure the trigger with the following settings:

      - **Name**: `devops-trigger`
      - **Region**: `europe-west4`
      - **Repository**: Click **CONNECT NEW REPOSITORY**.
          - Select **GitHub (Cloud Build GitHub App)** and click **Continue**.
          - Authenticate and select your `{your-github-username}/devops-repo`.
          - Accept the terms and click **Connect**.
      - **Branch**: `.*` (any branch)
      - **Configuration**: Cloud Build configuration file (yaml or json)
      - **Location**: Inline

4.  In the inline editor, replace the default content with the following YAML. This configuration builds the Docker image and tags it with the unique commit SHA.

    ```yaml
    steps:
      - name: 'gcr.io/cloud-builders/docker'
        args: ['build', '-t', 'europe-west4-docker.pkg.dev/qwiklabs-gcp-02-cae7e00e7a13/devops-repo/devops-image:$COMMIT_SHA', '.']
    images:
      - 'europe-west4-docker.pkg.dev/qwiklabs-gcp-02-cae7e00e7a13/devops-repo/devops-image:$COMMIT_SHA'
    options:
      logging: CLOUD_LOGGING_ONLY
    ```

5.  Click **Done** and then **Create**.

6.  To test the trigger, find your new trigger in the list, click **Run**, and then **Run trigger**. Check the **History** tab to see the build in progress.

7.  Now, let's test the automation. In the Cloud Shell Editor, open `main.py` and change the title in the `main()` function:

    ```python
    # main.py
    @app.route("/")
    def main():
        model = {"title": "Hello Build Trigger."}
        return render_template("index.html", model=model)
    ```

8.  Commit and push this change to trigger a new build automatically:

    ```sh
    cd ~/gcp-course/devops-repo
    git commit -a -m "Testing Build Trigger"
    git push origin main
    ```

9.  Go back to the **Cloud Build \> History** page in the console. You should see a new build running, triggered by your latest push.

## Task 6: Test Your Build Changes

Once the automated build is complete, deploy the new image to verify the changes.

1.  In the **Cloud Build \> History** page, click on the latest completed build.

2.  Click the **Execution Details** tab and find the **Image name** in the "Images built" section. Click the name to go to its page in Artifact Registry.

3.  At the top of the Artifact Registry page, click the **Copy** icon next to the full image path with the `sha256` digest. You will need this for the next step.

4.  Go to **Compute Engine \> VM Instances**. Create a new VM instance as you did before.

5.  When deploying the container, paste the full image path you just copied.

6.  Remember to check **Allow HTTP traffic**.

7.  Once the new VM is running, click its external IP address. The browser should now display your updated message: "Hello Build Trigger."
