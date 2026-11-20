FROM python:3.9-alpine

WORKDIR /app

COPY requirements.txt ./

# Create non-root user
RUN adduser -D play

# Install dependencies system-wide
RUN python -m pip install --no-cache-dir -r requirements.txt

# Verify dependencies during image build
RUN python -c "import flask; import requests"

# Copy application files
COPY --chown=play:play order_service/ /app/

USER play

EXPOSE 8083

CMD ["python", "order_app.py"]
