#!/usr/bin/env bash

command="${1}"
timeout="${2:-30}"

i=0
until eval "${command}"
do
    sleep 1
    ((i++))

    if [ "${i}" -gt "${timeout}" ]; then
        echo "command was never successful, aborting due to ${timeout}s timeout!"
        exit 1
    fi
done
