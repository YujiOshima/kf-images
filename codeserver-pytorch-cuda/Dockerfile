FROM public.ecr.aws/j1r0q0g6/notebooks/notebook-servers/codeserver-python:v1.3.1-rc.0
USER root
COPY cuda-requirements.txt /tmp/requirements.txt
RUN python3 -m pip install -r /tmp/requirements.txt --quiet --no-cache-dir \
    && rm -f /tmp/requirements.txt
USER $NB_UID

EXPOSE 8888

ENTRYPOINT ["/init"]