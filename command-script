#!/bin/bash

##
# Sends a Slack webhook containing a list of all
# the JIRA tickets since the last push.
##

## Environment variables used in the script
# 
# PRODUCT_NAME
#
# JIRA_PREFIX
# JIRA_USER_EMAIL
# JIRA_API_TOKEN
#
# SLACK_WEBHOOK_URL
# 
##

# Config
product_name=${PRODUCT_NAME:-Product}
timestamp=`date +%s`
slack_webhook_url="$SLACK_WEBHOOK_URL"
prev_push_commit_hash=`echo "$GITHUB_EVENT_PATH" | jq '.before'`

tmp_file_jira_tickets="tmp-jira-tickets-$timestamp.txt"
tmp_file_other_commits="tmp-other-commits-$timestamp.txt"
tmp_file_output="tmp-commits-$timestamp.txt"

# Get the number of commits since last push commit hash
num_commits=`git rev-list $prev_push_commit_hash..HEAD --count`

# Get list of all commits that contain a JIRA ticket number and a separate list of all other commits
git log --pretty=format:"%aN - <%h> - %s" -n $num_commits \
    | tee >(grep [A-Z]+-[0-9]+ -E > $tmp_file_jira_tickets) \
          >(grep -vo [A-Z]+-[0-9]+ -E > $tmp_file_other_commits) \
      > /dev/null


# Output a list of JIRA tickets with accompanying details, pulled from JIRA directly
echo "

Jira Tickets
---------------------" >> $tmp_file_output
if [ ! -f "$tmp_file_jira_tickets" ]; then
  echo "
  No JIRA tickets detected since last push." >> $tmp_file_output
fi

while read t ; \
do \
  ticket_no="$(echo "$t" | grep -o [A-Z]+-[0-9]+ -E)"
  ticket_json="$(curl --silent -u $JIRA_USER_EMAIL:$JIRA_API_TOKEN -H "Content-Type: application/json" https://$JIRA_PREFIX.atlassian.net/rest/api/2/issue/$ticket_no)"
  
  if [ -z "$ticket_json" ];
  then
    ticket_type=`echo "${ticket_json}" | jq '.fields.issuetype.name' | sed "s/\"//g"`
    ticket_title=`echo "${ticket_json}" | jq '.fields.summary'`

  echo "
  * <$ticket_type> - $ticket_title
  [$ticket_no] https://$JIRA_PREFIX.atlassian.net/browse/$ticket_no" >> $tmp_file_output ; \
  
  else
  
  echo "
  * <JIRA ERROR> - $t
  [$ticket_no] https://$JIRA_PREFIX.atlassian.net/browse/$ticket_no" >> $tmp_file_output ; \
  
  fi

done <$tmp_file_jira_tickets


# Print out the remaining other commits along with author details
if [ -f "$tmp_file_other_commits" ]; 
then

echo "

Other Commits
---------------------
" >> $tmp_file_output
while read t ; \
do \
  echo "  * $t" >> $tmp_file_output ; \
done <$tmp_file_other_commits

fi


# Send a Slack webhook
# header="> <!here> *$product_name has been updated!*\n>*JIRA Tickets Mentioned <!date^$timestamp^- {date_long_pretty} {time}|just now>*\n"
# payload=`< $tmp_file_output sed '/^[ \t]*$/d' | paste -sd "\\n" -`
# curl -X POST --data-urlencode "payload={\"text\": \"$header\n$payload\"}" $slack_webhook_url

# Cleanup, remove the temporary file
cat $tmp_file_output
rm $tmp_file_jira_tickets
rm $tmp_file_other_commits
rm $tmp_file_output
