
### VARIABLE SETUP ###

ARCH?=$(shell uname -m)
# Hidden directory to install dependencies for jenkins
export PATH := ./bin:$(PATH)
export GO15VENDOREXPERIMENT=1
GOFILES = $(shell find . -type f -name '*.go')
SHELL=/bin/bash
ES_HOST?="elasticsearch-210"
BEAT_DIR?=github.com/elastic/beats
BUILD_DIR=build
COVERAGE_DIR=${BUILD_DIR}/coverage
TIMEOUT?= 90
BEATNAME?=libbeat
TEST_ENVIRONMENT?=false
SYSTEM_TESTS?=false
GOX_OS?=linux darwin windows solaris freebsd netbsd openbsd


### BUILDING ###

# Builds beat
.PHONY: build
build: $(GOFILES)
	go build

# Create test coverage binary
.PHONY: libbeat.test
libbeat.test: $(GOFILES)
	go test -c -covermode=atomic -coverpkg ./...

# Cross-compile beat for the OS'es specified in GOX_OS variable.
# The binaries are placed in the build/bin directory.
.PHONY: crosscompile
crosscompile: $(GOFILES)
	go get github.com/mitchellh/gox
	mkdir -p ${BUILD_DIR}/bin
	gox -output="${BUILD_DIR}/bin/{{.Dir}}-{{.OS}}-{{.Arch}}" -os="${GOX_OS}" ${GOX_FLAGS}

# Checks project and source code if everything is according to standard
.PHONY: check
check:
	gofmt -l . | read && echo "Code differs from gofmt's style" && exit 1 || true
	go vet ./...

# Cleans up directory and source code with gofmt
.PHONY: clean
clean:
	go fmt ./...
	gofmt -w .
	-rm -r build
	-rm ${BEATNAME} ${BEATNAME}.test

# Shortcut for continuous integration
# This should always run before merging.
.PHONY: ci
ci:
	make
	make check
	make testsuite

### Testing ###
# All tests are always run with coverage reporting enabled


# Prepration for tests
.PHONY: prepare-tests
prepare-tests:
	mkdir -p ${COVERAGE_DIR}
	# gotestcover is needed to fetch coverage for multiple packages
	go get github.com/pierrre/gotestcover

# Runs the unit tests
# Race is not enabled for unit tests because tests run much slower.
.PHONY: unit-tests
unit-tests: prepare-tests
	$(GOPATH)/bin/gotestcover -tags=unit -coverprofile=${COVERAGE_DIR}/unit.cov -covermode=atomic ${BEAT_DIR}/${BEATNAME}/...

# Run integration tests. Unit tests are run as part of the integration tests. It runs all tests with race detection enabled.
.PHONY: integration-tests
integration-tests: prepare-tests
	$(GOPATH)/bin/gotestcover -tags=integration -race -coverprofile=${COVERAGE_DIR}/integration.cov -covermode=atomic ${BEAT_DIR}/${BEATNAME}/...

# Runs the integration inside a virtual environment. This can be run on any docker-machine (local, remote)
.PHONY: integration-tests-environment
integration-tests-environment:
	make prepare-tests
	make build-image
	make stop-environment
	NAME=$$(docker-compose run -d ${BEATNAME} make integration-tests | awk 'END{print}') || exit 1; \
	echo "docker ${BEATNAME} test container: '$$NAME'"; \
	docker attach $$NAME; CODE=$$?;\
	mkdir -p ${COVERAGE_DIR}; \
	docker cp $$NAME:/go/src/${BEAT_DIR}/${BEATNAME}/${COVERAGE_DIR}/integration.cov $(shell pwd)/${COVERAGE_DIR}/; \
	docker rm $$NAME > /dev/null; \
	exit $$CODE

# Runs the system tests
.PHONY: system-tests
system-tests: libbeat.test prepare-tests system-tests-setup
	. build/system-tests/env/bin/activate; nosetests -w tests/system --process-timeout=$(TIMEOUT) --with-timer
	python ../libbeat/scripts/aggregate_coverage.py -o ${COVERAGE_DIR}/system.cov ./build/system-tests/run

# Runs the system tests
.PHONY: system-tests-setup
system-tests-setup: tests/system/requirements.txt
	test -d env || virtualenv build/system-tests/env > /dev/null
	. build/system-tests/env/bin/activate && pip install -Ur tests/system/requirements.txt > /dev/null
	touch build/system-tests/env/bin/activate


# Run benchmark tests
.PHONY: benchmark-tests
benchmark-tests:
	# No benchmark tests exist so far
	#go test -short -bench=. ./...

# Runs all tests and generates the coverage reports
.PHONY: testsuite
testsuite:
	make unit-tests

	# Setups environment if TEST_ENVIRONMENT is set to true
	if [ $(TEST_ENVIRONMENT) = true ]; then \
		 make integration-tests-environment; \
	else \
		make integration-tests; \
	fi

	# Runs system tests if SYSTEM_TESTS is set to true
	if [ $(SYSTEM_TESTS) = true ]; then \
		 make system-tests; \
	fi

	make benchmark-tests
	make coverage-report

# Generates a coverage report from the existing coverage files
# It assumes that some covrage reports already exists, otherwise it will fail
.PHONY: coverage-report
coverage-report:
	python ${GOPATH}/src/github.com/elastic/beats/dev-tools/aggregate_coverage.py -o ./${COVERAGE_DIR}/full.cov ./${COVERAGE_DIR}
	go tool cover -html=./${COVERAGE_DIR}/full.cov -o ${COVERAGE_DIR}/full.html

# Update expects the most recent version of libbeat in the GOPATH
.PHONY: update
update:
	bash ${GOPATH}/src/github.com/elastic/beats/libbeat/scripts/update.sh ${BEATNAME} ${BEAT_DIR}/${BEATNAME} ${GOPATH}/src/github.com/elastic/beats/libbeat



### CONTAINER ENVIRONMENT ####

# Builds the environment to test beat
.PHONY: build-image
build-image: write-environment
	docker-compose build

# Runs the environment so the redis and elasticsearch can also be used for local development
# To use it for running the test, set ES_HOST and REDIS_HOST environment variable to the ip of your docker-machine.
.PHONY: start-environment
start-environment: stop-environment
	docker-compose up -d redis elasticsearch-173 elasticsearch-210 logstash

.PHONY: stop-environment
stop-environment:
	-docker-compose stop
	-docker-compose rm -f
	-docker ps -a  | grep ${BEATNAME} | grep Exited | awk '{print $$1}' | xargs -r docker rm

.PHONY: write-environment
write-environment:
	mkdir -p build
	echo "ES_HOST=${ES_HOST}" > build/test.env
	echo "ES_PORT=9200" >> build/test.env
