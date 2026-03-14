
## Creating environment using conda
conda create -p ./env python=3.11
conda activate env

## Creating environment using python
python -m venv venv
venv\Scripts\activate
source venv\bin\activate

## pip install -e . it tries find the setup.py or project.toml files

