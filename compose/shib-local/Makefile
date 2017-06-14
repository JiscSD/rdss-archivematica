.DEFAULT_GOAL := build

BASE_DIR ?= ${CURDIR}

VOL_BASE ?= $(BASE_DIR)/..

DEFAULT_COMPOSE_FILE = $(shell realpath ../docker-compose.shib-local.yml)

DOCKER_BUILD_GID ?= $(shell id -g)
DOCKER_BUILD_UID ?= $(shell id -u)

DOMAIN_NAME ?= example.ac.uk

all: destroy init-ca build create-secrets up bootstrap list

build: build-idp
	COMPOSE_FILE=$(DEFAULT_COMPOSE_FILE) docker-compose build

build-idp:
	# Create the build dir for the IdP, ensuring it's within the docker build context
	@docker run --rm \
		--user $(DOCKER_BUILD_UID):$(DOCKER_BUILD_GID) \
		--volume "$(VOL_BASE)/shib-local/:/build_ctx" \
		alpine \
		mkdir -p /build_ctx/build/idp
	# Generate IdP metadata-providers.xml
	@docker run --rm \
		--volume "$(VOL_BASE)/shib-local/idp-metadata-providers.py:/src/idp-metadata-providers.py:ro" \
		--volume "$(VOL_BASE)/shib-local/etc/idp/:/src/etc/:ro" \
		--volume "$(VOL_BASE)/shib-local/build/idp/:/build" \
		--workdir "/src" \
		python:2-alpine \
		python idp-metadata-providers.py /src/etc/service-providers.json > \
			"$(BASE_DIR)/build/idp/metadata-providers.xml"
	# Create base IdP configuration
	@docker run --rm \
		-h idp.$(DOMAIN_NAME) \
		--volume "$(VOL_BASE)/shib-local/idp/bin:/setup/bin:ro" \
		--volume "$(VOL_BASE)/shib-local/etc/idp:/setup/conf:ro" \
		--volume "$(VOL_BASE)/shib-local/build/idp/:/ext-mount" \
		-e DOMAIN_NAME=$(DOMAIN_NAME) \
		-e IDP_OWNER_UID=$(shell id -u) \
		-e IDP_OWNER_GID=$(shell id -g) \
		unicon/shibboleth-idp /setup/bin/init-idp.sh
	# Copy IdP dockerfile to build dir as part of its container context
	@docker run --rm \
		--user $(shell id -u):$(shell id -g) \
		--volume "$(VOL_BASE)/shib-local/:/build_ctx" \
		alpine \
		cp -p /build_ctx/idp/Dockerfile /build_ctx/build/idp/

# This is needed by upstream makefile
bootstrap:

create-secrets:
	# Create the secrets dir for the IdP, ensuring it's within the docker build context
	@docker run --rm \
		--user $(shell id -u):$(shell id -g) \
		--volume "$(VOL_BASE)/shib-local/:/build_ctx" \
		alpine \
		mkdir -p /build_ctx/build/secrets/idp
	# Create keys and certs for the IdP service
	@docker run --rm \
		--user $(shell id -u):$(shell id -g) \
		--volume "$(VOL_BASE)/shib-local/ca/:/src/ca" \
		--volume "$(VOL_BASE)/shib-local/idp/:/src/idp:ro" \
		--volume "$(VOL_BASE)/shib-local/build/secrets/idp:/build" \
		--env DOMAIN_NAME="$(DOMAIN_NAME)" \
		--workdir "/src/idp" \
		rawmind/alpine-base:3.5-1 \
		./create-secrets.sh

destroy:
	@mkdir -p "$(BASE_DIR)/build/idp/"
	# Stop containers and remove volumes
	COMPOSE_FILE=$(COMPOSE_FILE) docker-compose down --volumes

init-ca:
	# Initialise the Certificate Authority. We do this within a docker container
	# so that it happens on the Docker host, not the build machine.
	@docker run --rm \
		--user $(shell id -u):$(shell id -g) \
		--volume "$(VOL_BASE)/shib-local/ca:/ca" \
		--env DOMAIN_NAME="$(DOMAIN_NAME)" \
		--workdir "/ca" \
		rawmind/alpine-base:3.5-1 \
		/bin/bash -c "[ -f /ca/domains/$DOMAIN_NAME/certs/${DOMAIN_NAME}-ca.crt ] || ./init.sh"

nuke-ca:
	# Nuke the Certificate Authority (not necessary for every rebuild)
	@docker run --rm \
		--user $(shell id -u):$(shell id -g) \
		--volume "$(VOL_BASE)/shib-local/ca:/ca" \
		--env DOMAIN_NAME="$(DOMAIN_NAME)" \
		--workdir "/ca" \
		rawmind/alpine-base:3.5-1 \
		./nuke.sh

.PHONY: all bootstrap build destroy init-ca nuke-ca