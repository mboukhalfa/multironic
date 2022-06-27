#!/bin/bash

DIR="$(dirname "$(readlink -f "$0")")"

if [ -d "${PWD}/_clouds_yaml" ]; then
  MOUNTDIR="${PWD}/_clouds_yaml"
else
  echo "cannot find _clouds_yaml"
  exit 1
fi

if [ "$1" == "baremetal" ] ; then
  shift 1
fi

# shellcheck disable=SC2086
sudo podman run --net=host \
  -v "${MOUNTDIR}:/etc/openstack" --rm \
  -e OS_CLOUD="${OS_CLOUD:-metal3}" "http://10.201.10.192:5000/localimages/ironic-client" "$@"
