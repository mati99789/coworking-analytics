FROM public.ecr.aws/docker/library/python:3.11

WORKDIR /app


COPY requirements.txt .


RUN pip install --upgrade pip && \
    pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 5153

CMD [ "python", "app.py" ]