# ----------------------------------------------------------------------------------------------------------
# Watchdog Stage
#
# Used to enable the OpenFaas watchdog which is responsible for 
# starting and monitoring functions in OpenFaaS
# ----------------------------------------------------------------------------------------------------------

FROM openfaas/classic-watchdog:0.18.17 as watchdog

# ----------------------------------------------------------------------------------------------------------
# Build Stage
#
# using the latest slim rust as a builder
# ----------------------------------------------------------------------------------------------------------

FROM rust:1.43-slim as builder

# Allows you to add additional packages via build-arg
ARG ADDITIONAL_PACKAGE

COPY --from=watchdog /fwatchdog /usr/bin/fwatchdog
RUN chmod +x /usr/bin/fwatchdog

WORKDIR /home/app

# add tools for building cross-platform images
RUN apt-get update -qy \
    && apt-get install -qy ca-certificates pkg-config libssl-dev musl-tools ${ADDITIONAL_PACKAGE} --no-install-recommends
RUN rustup target add x86_64-unknown-linux-musl

COPY function ./function

# Rust's package manager, Cargo, does not have an option to build dependencies indepently meaning 
# typically everytime this file is run, all dependencies must be downloaded and compiled again.
# Since ALL dependencies must first be added in the Cargo.toml before using in source files, prevent 
# the dependencies rebuilding on source file changes (in /main/src) 
COPY main/Cargo.toml ./main/Cargo.toml

# create the source directory and add the required main.rs file, with valid "dummy" content
RUN mkdir main/src/
RUN echo "extern crate handler;\nfn main() {println!(\"if you see this, the build broke\")}" > main/src/main.rs
# Run cargo build (in release mode), which will check dependencie and download those that are missing
RUN RUSTFLAGS=-Clinker=musl-gcc cargo build --release --target=x86_64-unknown-linux-musl --manifest-path ./main/Cargo.toml

# now remove the dummy source files
RUN rm -rf main/
# remove the build output from dummy files (but preserve dependencies in /target directory!)
RUN rm -f /home/app/main/target/x86_64-unknown-linux-musl/release/main*
RUN rm -f /home/app/main/target/x86_64-unknown-linux-musl/release/deps/main*
RUN rm -f /home/app/main/target/x86_64-unknown-linux-musl/release/main.d

# copy the real source files and rebuild. Since dependencies are still present, no downloads are necessary 
COPY main ./main
RUN RUSTFLAGS=-Clinker=musl-gcc cargo build --release --target=x86_64-unknown-linux-musl --manifest-path ./main/Cargo.toml

# ----------------------------------------------------------------------------------------------------------
# Final Stage
#
# use a smaller image (e.g. Alpine) for the final output of the build
# ----------------------------------------------------------------------------------------------------------

FROM scratch

COPY --from=builder /usr/bin/fwatchdog .
COPY --from=builder /home/app/main/target/x86_64-unknown-linux-musl/release/main .

USER 1000

ENV fprocess="./main"

EXPOSE 8080

HEALTHCHECK --interval=3s CMD [ -e /tmp/.lock ] || exit 1

CMD ["./fwatchdog"]
