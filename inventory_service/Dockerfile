
FROM python:3.9-alpine

WORKDIR /app

COPY requirements.txt ./

RUN adduser -D play

RUN pip install --no-cache-dir --user -r requirements.txt

ENV PATH="/home/play/.local/bin:$PATH"

COPY --chown=play:play inventory_service/ /app/

USER play

EXPOSE 8082

CMD ["python", "inventory_app.py"]
