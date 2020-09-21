# Developing an AGENT in Golang

This document defines the steps to create an agent in [golang](https://golang.org). To build an agent the following steps should be followed:


1. Initialize the project using go mod
2. Yang
	1. Define the YANG model
	2. Validate the YANG model
	3. Generate Code for the YANG model



## Initialize the project using go mod

```
go mod init github.com/srl-wim/srl-ndk-git
```


## YANG

### Defining the YANG model

The first step in creating an agent is defining the YANG model, which defines the configuration and state of the application.

### Validate the yang model

Once you have defined the YANG model, we should check for errors and validate the model. pyang is a perfect tool for that as shown in the below example.

```
(srl) henderiw@henderiw-mackbook-pro-16 git % pyang -f tree -p yang/ yang/git.yang
module: git
  +--rw git!
     +--rw organization?   string
     +--rw repo?           string
     +--rw file?           string
     +--rw token?          string
     +--rw author?         string
     +--rw author-email?   string
     +--rw branch?         string
     +--rw action?         string
     +--ro oper-state?     srl_nokia-comm:oper-state
     +--ro statistics
        +--ro success?   srl_nokia-comm:zero-based-counter64
        +--ro failure?   srl_nokia-comm:zero-based-counter64
```

## Golang

### 

## YML file

## GIT

```
# This is an example goreleaser.yaml file with some sane defaults.
# Make sure to check the documentation at http://goreleaser.com
before:
  hooks:
    # You may remove this if you don't use go modules.
    - go mod download
    # you may remove this if you don't need go generate
    - go generate ./...
builds:
  - env:
      - CGO_ENABLED=0
    ldflags:
      - -s -w -X github.com/srl-wim/srl-ndk-git/cmd.version={{.Version}} -X github.com/srl-wim/srl-ndk-git/cmd.commit={{.ShortCommit}} -X github.com/srl-wim/srl-ndk-git/cmd.date={{.Date}}
    goos:
      - linux
      # - windows
      - darwin
archives:
  - replacements:
      darwin: Darwin
      linux: Linux
      #windows: Windows
      #386: i386
      #amd64: x86_64
checksum:
  name_template: "checksums.txt"
snapshot:
  name_template: "{{ .Tag }}-next"
changelog:
  sort: asc
  filters:
    exclude:
      - "^docs:"
      - "^test:"
nfpms:
  - id: "srlndk-git"
    package_name: "srlndk-git"
    maintainer: "Wim Henderickx <wim.henderickx@nokia.com>"
    description: |
      srlndk-git written in go
    vendor: "Nokia"
    license: "BSD 2"
    formats:
      - rpm
      - deb
    bindir: /usr/bin
    files:
      ./dist/srl-ndk-git_linux_amd64/srl-ndk-git: "/opt/srlinux/usr/bin/ndk-git"
    config_files:
      ./yang/ndk-git.yang: "/opt/ndk-git/yang/ndk-git.yang"
      ./yml/ndk-git.yml: "/etc/opt/srlinux/appmgr/ndk-git.yml"
    overrides:
      rpm:
        scripts:
```