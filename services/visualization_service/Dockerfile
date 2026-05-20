FROM python:3.11-slim

WORKDIR /app

RUN pip install --no-cache-dir fastapi==0.115.6 uvicorn[standard]==0.34.0 pydantic==2.10.4

COPY services/ services/

EXPOSE 8011

CMD ["uvicorn", "services.visualization_service.app:app", "--host", "0.0.0.0", "--port", "8011"]
