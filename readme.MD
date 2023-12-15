
#create a folder from terminal or manually from cpanel and make sure it has read and write permission
sudo mkdir -p /home/838162.cloudwaysapps.com/ngdxkwsbhc/public_html/3cx
sudo chown -R smaart: /home/838162.cloudwaysapps.com/ngdxkwsbhc/public_html/3cx
sudo chmod -R u+rw /home/838162.cloudwaysapps.com/ngdxkwsbhc/public_html/3cx

#Install following tools 

sudo apt-get update
sudo apt-get install sendmail diffutils coreutils grep


#Create a file 

touch 3cx_monitor.sh

#add following code to "3cx_monitor.sh" and make sure replace your smtp details 

#!/bin/bash

EMAIL_TO="your_email@example.com"
DIR="/home/838162.cloudwaysapps.com/ngdxkwsbhc/public_html/3cx"
LOGFILE="/var/log/3cx_monitor.log"
SENT_FILES_FILE="/var/log/3cx_sent_files.log"
CHECK_INTERVAL=60

# Option 1: Hardcoded SMTP details (uncomment and replace)
# SMTP_SERVER="your_smtp_server"
# SMTP_PORT="your_smtp_port"
# SMTP_USERNAME="your_smtp_username"
# SMTP_PASSWORD="your_smtp_password"

# Option 2: Use environment variables (set before running script)
# export SMTP_SERVER="your_smtp_server"
# export SMTP_PORT="your_smtp_port"
# export SMTP_USERNAME="your_smtp_username"
# export SMTP_PASSWORD="your_smtp_password"

required_tools=("sendmail" "diff" "basename" "grep")
for tool in "${required_tools[@]}"; do
  if ! command -v "$tool" &>/dev/null; then
    logger -t 3cx_monitor "Error: Missing required tool: $tool. Install it." >&2
    exit 1
  fi
done

function send_email {
  echo -e "$2" | sendmail -s "$1" "$EMAIL_TO" -v SMTPSERVER="$SMTP_SERVER" -v SMTPSERVERPORT="$SMTP_PORT" -v SMTPSERVERUSERNAME="$SMTP_USERNAME" -v SMTPSERVERPASSWORD="$SMTP_PASSWORD" --attach="$3" >> "$LOGFILE"
  echo "Email sent for $1."
}

if [[ ! -f "$SENT_FILES_FILE" ]]; then
  touch "$SENT_FILES_FILE"
fi

trap 'rm -f "$SENT_FILES_FILE"; exit 1' INT TERM EXIT

previous_files=""
while true; do
  current_files=$(ls -1 "$DIR")
  different_files=$(diff --newfile --unchanged --skip-empty-lines <(echo "$previous_files") <(echo "$current_files"))

  if [[ -n "$different_files" ]]; then
    for new_file in $different_files; do
      filename=$(basename "$new_file")
      timestamp=$(date +"%Y-%m-%d_%H-%M-%S")

      if [[ $(grep -q "$filename" "$SENT_FILES_FILE"; echo $?) -eq 0 ]]; then
        continue
      fi

      subject="[Record Added] - $filename"
      body="A new record has been added at $timestamp:\n\n$filename\n\nFor more details, please see the attached file."

      send_email "$subject" "$body" "$DIR/$new_file"

      echo "$filename" >> "$SENT_FILES_FILE"
    done
    previous_files="$current_files"
  fi

  sleep "$CHECK_INTERVAL"
done

