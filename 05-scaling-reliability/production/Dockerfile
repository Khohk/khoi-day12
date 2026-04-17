FROM python:3.11-slim

WORKDIR /app

COPY 05-scaling-reliability/production/app.py .
COPY 05-scaling-reliability/production/utils/ utils/

RUN pip install --no-cache-dir fastapi uvicorn redis python-dotenv

EXPOSE 8000

CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
