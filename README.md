# gae-source

Repository dedicated to explore the different options available to download the source code from Google App Engine after the deprecation of the `appcfg.py` tool, differentiating between Standard and Flexible options.

## Stackdriver Debugger

Applications that are deployed by using `gcloud beta app deploy` with gcloud version 195.0.0 or newer, will have the code available to inspect at Stackdriver Debugger. From there, even though not being 100% reliable and not providing any way of downloading the code, it is possible to extract it by copying and pasting 
As said, it is not 100% reliable and the latest code changes may not be showing, but still worth a try when there's no other option available.

## Staging Bucket
The TTL of the files is 15 days inside the Staging bucket, therefore is the more suitable option for latest updates or pipelines that are deploying to App Engine on a frequently manner. Opposed to Stackdriver Debugger, in here we have a programatic way of downloading files and not having to recur to the good old copy and paste. As we're fetching from a Cloud Storage bucket, `gsutil` it's available for that task:

### Simple Version
```
#Before anything else, list what's inside our bucket
gsutil ls gs://staging.$PROJECT_ID.appspot.com
```
Although not fully reliable as well, we can use deployment timestamps (or at least filter by a determined range of time) to look for all the files we're interested in. This approach will work unless there's matching in time deployments, which is quite unlikely to happen though.

```
gsutil ls -l gs://staging.$PROJECT_ID.appspot.com | awk -v s="2020-05-07T14:40:00Z" -v e="2020-05-07T14:50:00Z" 's<=$2 && $2<=e {print $3}' >> gae-source-code.txt

cat gae-source-code.txt | gsutil -m cp -I ./$DEST_DIRECTORY
```

### Advanced Version
If we take another look at our staging bucket, we will see a very special folder: `ae`. This contains the `manifest.json` which has a mapping of all the files and what are they exactly. With this approach we will be able to correctly name our files at the same time we're going to download them, as with the approach shown above, names are hashed and thus harder to guess what are them exactly. In a similar fashion as done above, let's filter out the manifest.jsons in the `ae` folder and look for the desired timestamp:

```
# Let's see our manifest.json files inside the ae folder, before we proceed.
gsutil ls -lr gs://staging.$PROJECT_ID.appspot.com/ae

# Interestingly enough, our code from above works here as well, and allow us to find the manifest.json we're looking for
gsutil ls -lr gs://staging.$PROJECT_ID.appspot.com/ae | awk -v s="2020-05-07T14:40:00Z" -v e="2020-05-07T14:50:00Z" 's<=$2 && $2<=e {print $3}' >> manifest-url.txt

# Let's download it so we can keep going:
cat manifest-url.txt | gsutil cp -I manifest.json
```
Now that we have our `manifest.json` file, let's process it so we can generate a `.txt` file as we did before, so we have each filename paired with their source URL. 
!! JQ needs to be installed ¡¡

```
jq 'to_entries[] | [.key, .value.sourceUrl] | @tsv' manifest.json | cut -f 2 -d'"' >> parsed-manifest.txt
sed -i 's/\\t/  /g' parsed-manifest.txt
sed -i 's#https\:\/\/storage\.googleapis\.com\/#gs\:\/\/#g' parsed-manifest.txt
while IFS= read -r line; do
    file=$(echo "$line" | awk '{print $1}')
    path=$(echo "$line" | awk '{print $2}')
    echo "Downloading $file ...."
    gsutil cp $path $file
done < parsed-manifest.txt
```

Now we have a 100% reliable method to ensure that the files we are downloading correspond to the same service, although the uncertainty of having filtered by the deployment timestamp still applies.

## VM SSHing

## Google Container Repository Image
Supported regions:
```
'gcr.io', 'us.gcr.io', 'eu.gcr.io', 'asia.gcr.io'
```

`gcloud container images list --repository $REGION.gcr.io/$PROJECT_ID/appengine`
