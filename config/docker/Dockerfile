FROM python:latest
RUN git clone --branch cli-hostname-binding-option https://github.com/tedhtchang/feast.git
RUN cd feast; make install-python;
RUN pip install feast[gcp] feast[redis] feast[aws]
ENV FEAST_USAGE=False
EXPOSE 6566
ENTRYPOINT ["feast"]
CMD ["--help"]

