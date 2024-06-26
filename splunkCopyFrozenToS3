#!/bin/bash
# Variables to set:
# AWS_ACCESS_KEY_ID
# AWS_SECRET_ACCESS_KEY
# AWS_REGION
# LOGFILE
# CURRENT_ENDPOINT_HOST
# FROZEN_BUCKET
# NEW_BUCKET_DIRECTORY
# UPLOADED_BUCKET_DIRECTORY
# LOGFILE

export AWS_ACCESS_KEY_ID=""
export AWS_SECRET_ACCESS_KEY=""
export AWS_REGION="snow"

CURRENT_ENDPOINT_HOST="0.0.0.0"
FROZEN_BUCKET="bucketname"
# Root directory to start the search
# Both paths should exist and be writable to Splunk. /data-freeze new should be the location data is frozen to.
NEW_BUCKET_DIRECTORY="/path/splunk/var/lib/splunk/frozenCache/new"
UPLOADED_BUCKET_DIRECTORY="/path/splunk/var/lib/splunk/frozenCache/uploaded"

LOGFILE="/var/log/splunkCopyFrozenToS3_$(date +%Y-%m-%d).log"
echo $(date)" splunkcopyFrozenToS3 routine starting." | tee -a $LOGFILE

CA_BUNDLE=$( cd -- "$( dirname -- "${BASH_SOURCE[0]}" )" &> /dev/null && pwd )/$CURRENT_ENDPOINT_HOST".pem"

if [ ! -f "$CA_BUNDLE" ]; then
  echo $(date)" Required CA Bundle not found.$CA_BUNDLE"| tee -a $LOGFILE
  exit 1
fi

#echo $CA_BUNDLE
ENDPOINT_URL=https://$CURRENT_ENDPOINT_HOST":8443"

# Uncomment the line below to test the endpoint.
# aws s3 ls s3://"$FROZEN_BUCKET"  --endpoint-url "$ENDPOINT_URL" --ca-bundle="$CA_BUNDLE" --region "$AWS_REGION"

# Bucket patterns to find.
ORIGINATED_BUCKET_PATTERN=".*db_[0-9]+_[0-9]+_[0-9]+_.*[^(rawdata)]"
REPLICATED_BUCKET_PATTERN=".*rb_[0-9]+_[0-9]+_[0-9]+_.*[^(rawdata)]"

# Initialize an array to hold matching directories
matching_dirs=()
# cd into bucket directory for relative paths
cd $NEW_BUCKET_DIRECTORY

while IFS= read -r dir; do
    matching_dirs+=("$dir")
done < <(find .  -regextype egrep -regex "$ORIGINATED_BUCKET_PATTERN" -type d)

# Print out the matching directories
for dir in "${matching_dirs[@]}"; do
#    echo "aws s3 cp $dir s3://$FROZEN_BUCKET/$(date +%Y-%m-%d)/$(hostname)/${dir#./} --recursive --endpoint-url "$ENDPOINT_URL"  --ca-bundle="$CA_BUNDLE"  --region "$AWS_REGION
    echo $(date)" Uploading "$(date +%Y-%m-%d)/$(hostname)/${dir#./} | tee -a $LOGFILE
    aws s3 cp $dir s3://$FROZEN_BUCKET/$(date +%Y-%m-%d)/$(hostname)/${dir#./} --recursive --endpoint-url "$ENDPOINT_URL" --ca-bundle="$CA_BUNDLE"  --region "$AWS_REGION"
    if [ $? -eq 0 ]
    then
      # get the local md5sum for local copy of the uploaded journal.zst file
      LOCAL_SIZE=$(stat -c%s "$dir"/rawdata/journal.zst)
      # get the md5sum for the uploaded journal.st file from aws
#      echo aws s3api head-object --bucket "$FROZEN_BUCKET" --key "$(date +%Y-%m-%d)/$(hostname)/${dir#./}/rawdata/journal.zst" --query ContentLength --output text  --endpoint-url "$ENDPOINT_URL" --ca-bundle="$CA_BUNDLE" --region "$AWS_REGION"
      S3_SIZE=$(aws s3api head-object --bucket "$FROZEN_BUCKET" --key "$(date +%Y-%m-%d)/$(hostname)/${dir#./}/rawdata/journal.zst" --query ContentLength --output text --endpoint-url "$ENDPOINT_URL" --ca-bundle="$CA_BUNDLE" --region "$AWS_REGION" )

      echo $(date)" Verifying upload size match. Local:"$LOCAL_SIZE" -eq AWS:"$S3_SIZE | tee -a $LOGFILE
      if [ "$LOCAL_SIZE" -eq  "$S3_SIZE" ]
      then
        mkdir -p $UPLOADED_BUCKET_DIRECTORY/$(date +%Y-%m-%d)/$(hostname)/$(basename $(dirname $dir))
        mv $dir -t $UPLOADED_BUCKET_DIRECTORY/$(date +%Y-%m-%d)/$(hostname)/$(basename $(dirname $dir))
        echo $(date)" Frozen files moved to uploaded directory." | tee -a $LOGFILE
      else
        echo $(date)" Size mistmatch, frozen files not moved." | tee -a $LOGFILE
      fi
    else
      echo $(date)" Failure during upload." | tee -a $LOGFILE
    fi
done

# Delete replicated buckets.
cd $NEW_BUCKET_DIRECTORY
find .  -regextype egrep -regex "$REPLICATED_BUCKET_PATTERN" -type d -exec rm -rf {} +

echo $(date)" splunkcopyFrozenToS3 routine finished." | tee -a $LOGFILE
