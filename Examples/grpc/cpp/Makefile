# Build Redis as follows:
#
# - make               -- create non-SGX no-debug-log manifest
# - make SGX=1         -- create SGX no-debug-log manifest
# - make SGX=1 DEBUG=1 -- create SGX debug-log manifest
#
# Any of these invocations clones Redis' git repository and builds Redis in
# default configuration and in the latest-to-date (5.0.5) version.
#
# By default, Redis uses poll/epoll mechanism of Linux. To build Redis with
# select, use `make USE_SELECT=1`. For correct re-builds, always clean up
# Redis source code beforehand via `make distclean`.
#
# Use `make clean` to remove Graphene-generated files and `make distclean` to
# additionally remove the cloned Redis git repository.

################################# CONSTANTS ###################################

# Relative path to Graphene root
GRAPHENEDIR = ../../..
SGX_SIGNER_KEY ?= $(GRAPHENEDIR)/Pal/src/host/Linux-SGX/signer/enclave-key.pem
MY_INSTALL_DIR ?= $(PWD)/.local
export PATH := $(MY_INSTALL_DIR)/bin:$(PATH)

ifeq ($(DEBUG),1)
GRAPHENE_LOG_LEVEL = debug
else
GRAPHENE_LOG_LEVEL = error
endif

.PHONY: all
all: grpc-server grpc-server.manifest
ifeq ($(SGX),1)
all: grpc-server.manifest.sgx grpc-server.sig grpc-server.token
endif

include ../../../Scripts/Makefile.configs

################################ gRPC MANIFEST ###############################

# The template file contains almost all necessary information to run grpc 
# under Graphene / Graphene-SGX. We create grpc-server.manifest (to be run under
# non-SGX Graphene) by simply replacing variables in the template file via sed.

grpc-server.manifest: grpc-server.manifest.template
	graphene-manifest \
		-Dlog_level=$(GRAPHENE_LOG_LEVEL) \
		-Darch_libdir=$(ARCH_LIBDIR) \
		$< > $@

# Manifest for Graphene-SGX requires special "graphene-sgx-sign" procedure. This procedure measures
# all Redis trusted files, adds the measurement to the resulting manifest.sgx file (among other,
# less important SGX options) and creates redis-server.sig (SIGSTRUCT object).
#
# Graphene-SGX requires EINITTOKEN and SIGSTRUCT objects (see SGX hardware ABI, in particular EINIT
# instruction). The "graphene-sgx-get-token" script generates EINITTOKEN based on a SIGSTRUCT and
# puts it in .token file. Note that filenames must be the same as the manifest name (i.e.,
# "redis-server"). EINITTOKEN must be generated on the machine where the application will run, not
# where it was built.

# Make on Ubuntu <= 20.04 doesn't support "Rules with Grouped Targets" (`&:`),
# we need to hack around.
grpc-server.sig grpc-server.manifest.sgx: sgx_outputs
	@:

.INTERMEDIATE: sgx_outputs
sgx_outputs: grpc-server.manifest
	graphene-sgx-sign \
		--key $(SGX_SIGNER_KEY) \
		--manifest grpc-server.manifest \
		--output grpc-server.manifest.sgx

grpc-server.token: grpc-server.sig
	graphene-sgx-get-token --output $@ --sig $<

########################### COPIES OF EXECUTABLES #############################

# Redis build process creates the final executable as src/redis-server. For
# simplicity, copy it into our root directory.

grpc-server: .local/bin/grpc_cpp_plugin
	cd grpc/examples/cpp/helloworld && mkdir -p cmake/build && cd cmake/build && \
	cmake -DCMAKE_PREFIX_PATH=$(MY_INSTALL_DIR) ../.. && make -j
	cp ./grpc/examples/cpp/helloworld/cmake/build/greeter_server grpc-server

.local/bin/grpc_cpp_plugin: grpc
	mkdir -p $(MY_INSTALL_DIR)
	cd grpc && mkdir -p cmake/build && cd cmake/build && \
	cmake -DgRPC_INSTALL=ON -DgRPC_BUILD_TESTS=OFF -DCMAKE_INSTALL_PREFIX=$(MY_INSTALL_DIR) ../.. && \
	make -j && make install
	cd grpc && mkdir -p third_party/abseil-cpp/cmake/build && cd third_party/abseil-cpp/cmake/build && \
	cmake -DCMAKE_INSTALL_PREFIX=$(MY_INSTALL_DIR) -DCMAKE_POSITION_INDEPENDENT_CODE=TRUE ../.. && \
	make -j && make install

grpc:
	git clone --recurse-submodules -b sgx-ratls https://github.com/Ruide/grpc

############################## RUNNING TESTS ##################################
.PHONY: start-native-server
start-native-server: all
	./grpc-server 

ifeq ($(SGX),)
GRAPHENE = graphene-direct
else
GRAPHENE = graphene-sgx
endif

.PHONY: start-graphene-server
start-graphene-server: all
	$(GRAPHENE) grpc-server 

################################## CLEANUP ####################################

.PHONY: clean
clean:
	$(RM) *.token *.sig *.manifest.sgx *.manifest grpc-server

.PHONY: distclean
distclean: clean
	$(RM) -rf grpc .local

################################## RUN ####################################
run:
	graphene-sgx grpc-server
	
