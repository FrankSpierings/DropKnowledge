### SOLR
A quick way to search documents for interesting data (read; passwords & usernames), is to use Solr. I've recently used this, by using a Docker container.

### Docker
1. Define some variables

```shell
MOUNTDIR=/tmp/mounts
CONTAINERNAME=mysolr
COLLECTOR=COLLECTOR1
```

2. Map the file location into the container. Example: We connected to a share through cifs on the host machine.

```shell
SERVERDIR="${MOUNTDIR}"/share1
mkdir -p "${SERVERDIR}"
mount -t cifs //server/someinterestingshare "${SERVERDIR}" -o username=stupiduser,password=stupidpassword
```

3. Now we create a Solr container including the mapping.

```shell
docker run --name "${CONTAINERNAME}" -v "${MOUNTDIR}":"${MOUNTDIR}":Z -d -p 8983:8983 -t solr
```

4. We create a new Solr `collector`

```shell
docker exec -it --user=solr "${CONTAINERNAME}" bin/solr create_core -c "${COLLECTOR}"
```

5. We modify the collector, so it will show the content stored in the `_text_` field (this is the content of a document).

```shell
curl -X POST -H 'Content-type:application/json' \
    --data-binary '{"replace-field":{"name":"_text_","type":"text_general","stored":true}}' \
    "http://localhost:8983/solr/${COLLECTOR}/schema"
```

6. We index the files in the mapped directory. (Don't want to wait? Issue `curl http://localhost:8983/solr/${COLLECTION}/update?commit=true`

```shell
docker exec -it --user=solr "${CONTAINERNAME}" bin/post -c "${COLLECTOR}" \
    -filetypes xml,json,csv,pdf,doc,docx,ppt,pptx,xls,xlsx,odt,odp,ods,ott,otp,ots,rtf,htm,html,txt,log,ini,xml,conf,config \
    "${MOUNTDIR}"
```

7. Now we can ask questions through `curl` or a webbrowser.

```shell
curl "http://localhost:8983/solr/${COLLECTOR}/select?q=password&wt=json&indent=on"
```


Examples:
- http://localhost:8983/solr/COLLECTOR1/query?q=password
- http://localhost:8983/solr/COLLECTOR1/query?q=admin&wt=xml
- http://localhost:8983/solr/COLLECTOR1/query?q=admin&wt=xml&fl=id


### References:
- https://cwiki.apache.org/confluence/display/solr/Common+Query+Parameters
