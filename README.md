Dependences to be installed
====================================

sudo apt install python3 python3-pip wget unzip git -y
python3 -m pip install --upgrade setuptools
python3 -m pip install --upgrade pip
python3 -m pip install PyMySQL
python3 -m pip install mysql-connector-python
python3 -m pip install psycopg2==2.7.5 --ignore-installed

If you get the following error `error: subprocess-exited-with-error` then run
`sudo apt install libpq-dev python3-dev build-essential`