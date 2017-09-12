### SOLR
A quick way to search documents for interesting data (read; passwords & usernames), is to use Solr. I've recently used this, by using a Docker container.

### Docker
1. Map the file location into the container. Example: We connected to a share through cifs on the host machine.
E.g:
```shell
mkdir /media/share
mount -t cifs //server/someinterestingshare /media/share -o username=stupiduser,password=stupidpassword
```

2. Now we create a Solr container including the mapping.
E.g:
```shell
docker run --name MYSOLR -v /media/share:/media/share:Z -d -p 8983:8983 -t solr 
```

3. We create a new Solr `collector`
E.g:
```shell
docker exec -it --user=solr MYSOLR bin/solr create_core -c COLLECTOR1 
```

4. We index the files in the mapped directory
E.g:
```shell
docker exec -it --user=solr MYSOLR bin/post -c COLLECTOR1 /media/share
```

Now we can ask questions through `curl` or a webbrowser.

Examples:
http://localhost:8983/solr/COLLECTOR1/query?q=password
http://localhost:8983/solr/COLLECTOR1/query?q=admin&wt=xml
http://localhost:8983/solr/COLLECTOR1/query?q=admin&wt=xml&fl=id


### References:
https://cwiki.apache.org/confluence/display/solr/Common+Query+Parameters