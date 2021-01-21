# djangorestframework_plus_elasticsearch
Helper to plug ElasticSearch on your API

This is a quick documantion to help pythoners plug ElasticSearch on their Restful API.
This can become a library but it will be depending on Janus, the god of the time and future.

### Script to load previous data into ES

```python
#/main/management/commands/es_load_data.py
import json
import requests
from decouple import config
from django.core.management.base import BaseCommand
from registration.signals import get_participant_serializer_data, get_note_serializer_data
from main.utilities.elasticsearch import insert_bulk, my_json_dumps
class Command(BaseCommand):
    help = 'Load data into elasticsearch configured on the .env file'
    def add_arguments(self, parser):
        parser.add_argument('-P', '--participants', action='store_true', help='Load Participants')
        parser.add_argument('-N', '--notes', action='store_true', help='Load Notes')
    def handle(self, *args, **options):
        op_all = options['participants'] is False and options['notes'] is False
        op_part = op_all or options['participants']
        op_not = op_all or options['notes']
        indexes = []
        if op_part:
            indexes.append({"index_name": "list_participants", "data": get_participant_serializer_data()})
        if op_not:
            indexes.append({"index_name": "list_notes", "data": get_note_serializer_data()})
        for index in indexes:
            data = index['data']
            index_name = index['index_name']
            node_name = index['index_name'].split('list_')[-1]
            es_endpoint = '%s/%s' % (index_name, node_name)
            # DOC: Create a file
            self.create_file(index_name, data)
            # DOC: Delete previous docs
            requests.delete('%s/%s' % (config('ELASTICSEARCH_API'), index_name))
            # DOC: Insert
            insert_bulk(es_endpoint, '@%s.json' % index_name)
            self.stdout.write(self.style.SUCCESS('[Success] %s data inserted in %s' %
                                                 (len(data), config('ELASTICSEARCH_API').split('@')[-1])))
    def create_file(self, index_name, data):
        file_name = '%s.json' % index_name
        with open(file_name, 'w') as f:
            for item in data:
                json_item = my_json_dumps(item)
                f.write('{"index":{"_id":"%s"}}' % json.loads(json_item)['id'])
                f.write('\n')
                f.write(json_item)
                f.write('\n')
        f.close()
        self.stdout.write(self.style.SUCCESS('[Success] %s data wrote in %s' % (len(data), file_name)))
        return file_name
```

### Helpers

```python
# main/utilities/elasticsearch.py
import os
import json
from decouple import config
from rest_framework.exceptions import ValidationError
from django.db.models.query import QuerySet
import requests
import logging
logger = logging.getLogger(__name__)
logger.setLevel(logging.WARNING)
# FIXME: Fix notes serializer: does it really need created_by field?
def set_default(obj):
    if isinstance(obj, set):
        return list(obj)
    if isinstance(obj, QuerySet):
        return [group for group in obj]
    raise TypeError
def my_json_dumps(query):
    return json.dumps(query, default=set_default)
def upsert_a_doc(item_data, feed):
    query = {
        "doc": item_data,
        "doc_as_upsert": True
    }
    json_data = my_json_dumps(query)
    url = '%s/%s?refresh=true' % (config('ELASTICSEARCH_API'), feed)
    response = requests.post(url, data=json_data, headers={'content-type': 'application/json'})
    if (response.status_code != 200):
        result = json.loads(response.text)
        if (result.get('error', None) is not None):
            msg = ' '.join([error['reason'] for error in result.get('error').get('root_cause')])
            raise ValidationError('[ES] ' + msg)
def insert_bulk(index_slash_doc_name, file_name):
    cmd = ("curl -s -H 'Content-Type: application/json' -XPOST '%s/%s/_bulk' --data-binary '%s'" %
           (config('ELASTICSEARCH_API'), index_slash_doc_name, file_name))
    out = os.popen(cmd).read()
    logger.warning('[ES.insert_bulk] %s' % out)
    return out
```

>>>>
```python
from rest_framework.response import Response
from django.dispatch import receiver
from main.utilities.elasticsearch import upsert_a_doc
from registration.serializers import ParticipantListSerializer
from registration.serializers.note import NoteSerializer
from registration.models import Participant, Note, CustomUser
from registration.permissions.utils import set_default_view_permissions

def update_elasticsearch_for_participant(sender, instance, created, **kwargs):
    return
    if (instance.__class__.__name__ == 'Participant'):
        participant_list = [instance]
    else:  # ESUpdateQuerySet
        if not isinstance(instance, list):
            instance = [instance]
        participant_list = [a_part.participant for a_part in instance]
    data = get_participant_serializer_data(queryset=participant_list)
    for item in data:
        upsert_a_doc(item, 'list_participants/_update/%s' % item['id'])
def get_participant_serializer_data(queryset=None):
    if (queryset is None):
        queryset = Participant.objects.all()
    serializer = ParticipantListSerializer(queryset, many=True)
    response = Response(serializer.data)
    return response.data
# ===================== Class Note =======================
def update_elasticsearch_for_note(sender, instance, created, **kwargs):
    data = get_note_serializer_data([instance])
    for item in data:
        upsert_a_doc(item, 'list_notes/_update/%s' % item['id'])
def get_note_serializer_data(queryset=None):
    if (queryset is None):
        queryset = Note.objects.all()
    serializer = NoteSerializer(queryset, many=True)
    response = Response(serializer.data)
    return response.data
```

>>>>>.env
```
...
ELASTICSEARCH_API=http://localhost:9200
...
```

### Dockerizing

```shell
# docker-compose.yml
  elasticsearch:
    container_name: elasticsearch
    image: docker.elastic.co/elasticsearch/elasticsearch:7.9.1
    environment:
      - "discovery.type=single-node"
      - "http.host=0.0.0.0"
      - "http.cors.enabled=true"
      - "http.cors.allow-origin=*"
      - "http.cors.allow-methods=OPTIONS, HEAD, GET, POST, PUT, DELETE"
      - "xpack.security.enabled=false"
      - "http.cors.allow-headers=Access-Control-Allow-Origin,x-csrf-token,authorization,content-type,accept,origin,x-requested-with"
    ports:
      - target: 9200
        published: 9200
      - target: 9300
        published: 9300
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    cap_add:
      - IPC_LOCK
    volumes:
      - elasticsearch-data:/usr/share/elasticsearch/data
    networks:
      - web
      
volumes:
  elasticsearch-data:
    driver: local
```

## Signing on Updating 

*Creating update signal*
```python
# /registration/utilities.py

# DOC: Explicit invokes a signal that updates elasticsearch documents
#   Due serializers that customizes the update method and eventually calls for the update() method instead of save()
#   --> Update method doesn't call signals in Django

post_update = Signal()
class ESUpdateQuerySet(QuerySet):
    def update(self, *args, **kwargs):
        super().update(*args, **kwargs)
        post_update.send(sender=self.model, instance=self, created=False)
```
*Calling update*
```python
# registration/models/remitter.py

from registration.utilities import ESUpdateQuerySet
class Remitter(models.Model):
    objects = ESUpdateQuerySet.as_manager()
...
```
