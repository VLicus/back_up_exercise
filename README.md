# Backup and Restore Automation Exercise

## Overview
The focus of this repository is on understanding backup types (full, incremental, and differential) and utilizing command-line interface (CLI) tools for backup and restore operations in MySQL and MongoDB databases. Additionally, automation of these processes through Bash scripting is implemented, and the scripts are scheduled using cron jobs for periodic backups.

## Contents

### 1. Backup Types
The documentation explains the concepts of full, incremental, and differential backups.

### 2. CLI Backup and Restore
Detailed instructions and examples are provided for using CLI tools to perform backups (full and incremental) and restores for both MySQL and MongoDB databases.

### 3. Bash Script Automation
Bash scripts are created to automate the backup and restore processes for either MySQL or MongoDB. The chosen database is configured for automated full backups once per week and incremental backups every day.

### 4. Cron Job Scheduling
Instructions for scheduling the Bash scripts using cron jobs are outlined. The schedule involves performing a full backup once per week and an incremental backup every day.

## Usage
Follow the step-by-step instructions in each section to understand backup concepts, perform CLI operations, and automate backups using Bash scripts. Adjust the cron job schedule as needed.
