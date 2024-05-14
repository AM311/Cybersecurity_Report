# Cybersecurity Hacking Lab: Report

This document will present a complete and step-by-step guide to perform a hacking laboratory activity in a virtualized environment, reproducing a fake but realistic scenario.

## Introduction to the activity

### Scenario

This laboratory tries to give an overview of the steps that might be performed by an attacker in a common organization running **Windows Active Directory** in order to take control of an high-privilege account.

The main characteristics of the hypotetical scenario are the following:

 - All the computers and devices of the organization are part of a **domain**, based on **Windows Active Directory**;
 - The domain mainly relies on two (physically and logically) different servers:
	 - A server that acts as **Domain Controller** and **DHCP server**;
	 - Another server that acts as **DNS server** and **File Server**;
 - Domain accounts are divided in two major "groups":
	 - **Domain Administrators**, which have "high privileges" on *all* machines;
	 - **Domain users**, which have "low privileges" on *all* machines;
 - Also, the following restrictions are enforced:
	 - **Domain Administrators** are set up to be also **Local Administrators** on *all* the machines;
	 - T

STRUTTURA GENERALE

DEBOLEZZE NELL'INFRASTRUTTURA

REQUISITI MINIMI (ASSUNTI SU CUI SI BASANO LE SEGUENTI AZIONI)


### Goal
The main goal of this cybersecurity laboratory activity is to gain access to the credentials of a domain administrator account.

### Threat Model
ddd

## Setting up the Virtual Environment

### Installing the Virtual Machines

This cybersecurity laboratory activity requires the set-up of a virtual environment, as better explained below.
Just for reference
The following virtual machines are required:

 - **2** VMs running **Windows Server 2012**;
 - **1** VM running **Windows 7**;
 - **1** VM running **Kali Linux**;

DETTAGLI TECNICI SU CONFIGURAZIONI VIRTUALBOX E VERSIONI SOFTWARE

RIMANDO A CONFIGURAZIONI COME DA SCENARIO --> CITARE PRINCIPALI MODI PER REALIZZARE LO SCENARIO

Just for reference, the **VirtualBox** virtualizator has been used for completing this activity.

## Let's start Hacking!


### Creating a local copy of the repository

Before we can work locally, we will need to create a clone of the repository.

When you clone a repository you are creating a copy of everything in that repository, including its history. This is one of the benefits of a DVCS like git - rather than being required to query a slow centralized server to review the commit history, queries are run locally and are lightning fast.

Let's go ahead and clone the class repository to your local desktop.

1. Navigate to the **Code** tab of the class repository on GitHub.
1. Click the green **Code** button.
1. Copy the **clone URL** to your clipboard.
1. Open your command-line application.
1. Retrieve a full copy of the repository from GitHub: `git clone <CLONE-URL>`
1. Once the clone is complete, navigate to the new directory created by the clone operation: `cd <REPOSITORY-NAME>`

### Our favorite Git command: `git status`

```shell-session
git status
On branch main
Your branch is up-to-date with 'origin/main'.
nothing to commit, working tree clean
```

`git status` is a command to verify the current state of your repository and the files it contains. Right now, we can see that we are on branch main, everything is up-to-date with origin/main and our working tree is clean.
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE3ODMyMjkyODEsLTQ3Mjg2OTkzNywtMT
I0NzcwNjkxMV19
-->