# syntax=docker/dockerfile:1
# ──────────────────────────────────────────────────────────────
# Custom geolab image -- built on pangeo/ml-notebook
#
# Unlike images built directly FROM pangeo/base-image, the parent's
# ONBUILD triggers do NOT fire again here (they already ran when
# ml-notebook itself was built). So apt.txt / environment.yml /
# requirements.txt must be applied explicitly, which is what this
# Dockerfile does.
#
# Build:
#   docker build --platform linux/amd64 \
#     --build-arg IMAGE_TITLE=my-geolab-image \
#     --build-arg IMAGE_AUTHORS=you@university.edu \
#     -t my-geolab-image .
# ──────────────────────────────────────────────────────────────
FROM pangeo/ml-notebook:latest

ARG IMAGE_TITLE="custom-geolab-image"   # default if user passes nothing
ARG IMAGE_AUTHORS="NoSpecifiedAuthors"  # default if user passes nothing

LABEL org.opencontainers.image.title="${IMAGE_TITLE}" \
      org.opencontainers.image.authors="${IMAGE_AUTHORS}"

# Set locations of PROJ/GDAL resource directories
ENV PROJ_DATA=/srv/conda/envs/notebook/share/proj \
    PROJ_LIB=/srv/conda/envs/notebook/share/proj \
    GDAL_DATA=/srv/conda/envs/notebook/share/gdal

# ── 1. apt packages (apt.txt) ─────────────────────────────────
# Needs root. xargs feeds each line of apt.txt to apt-get install.
USER root
COPY apt.txt /tmp/apt.txt
RUN echo "Installing apt packages from apt.txt..." \
    && apt-get update --fix-missing > /dev/null \
    && xargs -a /tmp/apt.txt apt-get install -y --no-install-recommends \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* /tmp/apt.txt

# Drop back to the non-root notebook user for conda/pip installs
USER ${NB_USER}

# ── 2. conda packages (environment.yml) ───────────────────────
# The env "notebook" already exists in the base image, so we
# UPDATE it rather than create it. Cleanup mirrors what the
# pangeo ONBUILD step does, to keep the image small.
COPY --chown=${NB_USER}:${NB_USER} environment.yml /tmp/environment.yml
RUN echo "Updating conda env 'notebook' from environment.yml..." \
    && mamba env update --name ${CONDA_ENV} -f /tmp/environment.yml \
    && mamba clean -yaf \
    && rm -f /tmp/environment.yml \
    && find ${CONDA_DIR} -follow -type f -name '*.a' -delete \
    && find ${CONDA_DIR} -follow -type f -name '*.js.map' -delete

# ── 3. pip packages (requirements.txt) ────────────────────────
# Installed into the same notebook env, after conda packages.
COPY --chown=${NB_USER}:${NB_USER} requirements.txt /tmp/requirements.txt
RUN echo "Installing pip packages from requirements.txt..." \
    && ${NB_PYTHON_PREFIX}/bin/pip install --no-cache-dir -r /tmp/requirements.txt \
    && rm -f /tmp/requirements.txt

# Default command for standalone use. JupyterHub spawning passes
# its own command (jupyterhub-singleuser), which /srv/start respects.
CMD ["jupyter", "lab", "--ip=0.0.0.0", "--no-browser"]