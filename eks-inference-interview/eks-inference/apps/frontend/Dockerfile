# Pin the base to an approved digest in a release build.
FROM python:3.12-slim AS build
WORKDIR /build
COPY requirements.lock .
RUN pip install --require-hashes --prefix=/install -r requirements.lock

FROM python:3.12-slim AS runtime
ENV PYTHONUNBUFFERED=1 PYTHONDONTWRITEBYTECODE=1
WORKDIR /app
COPY --from=build /install /usr/local
COPY --chown=10001:10001 src/ .
USER 10001:10001
CMD ["python", "-m", "server"]
EXPOSE 8080
