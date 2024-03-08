FROM python:3.10

WORKDIR /app

EXPOSE 5000

# Install pip requirements
COPY . /app
RUN python -m pip install -r requirements.txt

CMD ["python", "api.py"]