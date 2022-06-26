In this tutorial Django project runs in development and production environments.
I also show how to run a Django project using a WSGI server (Gunicorn) and a web server (Nginx).

## Step 1.  Create virtual environment using Anaconda 

```
curl -O https://repo.anaconda.com/archive/Anaconda3-2021.11-Linux-x86_64.sh

sha256sum Anaconda3-2021.11-Linux-x86_64.sh

bash Anaconda3-2021.11-Linux-x86_64.sh

```

Once finished, activate the installation by entering:
```
source ~/.bashrc

conda --version or  /home/ubuntu/anaconda3/condabin/conda --version
```
### Create virtual environment

```
conda create --name <name of env> <python=version>
```
Active newly created virtual environent
```
source active <name of virtual environment>
```

To Deactive the environment
```
source deactivate
```


## Step 2. Install django and gunicorn

```
pip install django
```
```
pip install gunicorn
```


