# Anonymize Request
anonymization happens by way of a series of celery tasks defined in [main/tasks.py](../sendit/apps/main/tasks.py) that are triggered by the first task that adds a complete set of dicom files (beloning to a dataset with one accession number) to the database as a batch. If you aren't familiar with the terms study, session, and how it relates to the radiologist workflow, the head of engineering at SOM IRT explained it well:

 - when a patient is placed on the scanner bed and the technician starts taking pictures, that’s a `study`.  Studies are identified by accession numbers and have a single associated date, patient, and ordering provider.
 - Each study may have more than one `Series` of images.  A series is what it sounds like: a set of closely associated images in a specified order.  A single image in a series is sometimes referred to as a `Slice`.  The PACS workstations allow you to rapidly scroll back and forth through all slices of a given study, just like a flip book, so you can "fly" through the scanned section of a patient’s body
 - After the conclusion of a study, a radiologist reviews the series' images and produces a report, referenced by the study accession number.  This accession number is present in the report and is a frequently used lookup key to get back to both the images and the associated report.

To be clear, for the purposes of this application a `batch` corresponds with one `study`, which is one or more series of images.  We will send one request to the DASHER API per batch (study), and then use the response to anonymize all associated images.


The tasks include the following:

## Get Identifiers
The first task `get_identifiers` under [main/tasks](sendit/apps/main/get.py) takes in a batch ID, and uses that batch to look up images, and flag them for the start of processing:

```
batch = Batch.objects.get(id=bid)
dicom_files = batch.get_image_paths()
batch.change_images_status('PROCESSING')
```

### 1. Extraction
We now want to extract all fields, including unwrapping nested lists, and we will do that with the function  `get_identifiers`. This function only skips over returning pixel data. This means that all header fields with a defined value (not None or blank) are returned. Items in sequences are unwrapped. Private values, which are usually for specific machines or applications, are not returned, and will be removed later in the anonymization process.


```
# deid get_identifiers: returns ids[entity][item] = {"field":"value"}
from deid.dicom import get_identifiers
ids = get_identifiers(dicom_files=dicom_files,
                      expand_sequences=True)  # expand sequences to flat structure
```

This returns a relatively flat data structure, ids, with a lookup for entity and then item. Eg:

```
ids[entity][item] = {"field1": "value1", ... , "fieldN": "valueN"}

```

In order to give each entity and item the unique IDs shown above (`entity` and `item`), we use default header fields `AccessionNumber` and `SOPInstanceUID`. These defaults are defined in the deid `dicom/config.json` file. If you want to change this in the sendit application, the `get_identifiers` (imported as `get_ids`) can take an optional `entity_id="CustomHeaderID"` and `item_id="CustomItemID"` fields. And also notice by default we expand sequences, meaning each item in any list (recursively to the deepest level) of items shoved into a single dicom field will be flattened into the structure above. 

### 2. Prepare Request
Once we have this flat list, we then need to prepare a very minimal request to send to the DASHER API, which expects a particular format, and specific fields related to the ids and timestamps of each item and entity. For DASHER, the preparation of that request looks like this:

```
request = prepare_identifiers_request(ids) # entity_custom_fields: True
                                           # item_custom_fields: False 
```

Note the defaults of the function are mentioned - by default we are including entity custom fields, but not item custom fields. We set these defaults by way of setting the variables `entity_custom_fields` to True (meaning we send custom entity fields to DASHER and `item_custom_fields` to False (meaning we don't send item custom fields). We originally were sending all of this data to the som DASHER endpoint, primarily with an entity id and timestamp, and then a huge list of `custom_fields` for each item and entity. This was a very slow process, and for purposes of searching, it puts a huge burden on DASHER for doing tasks outside of simple identity management. We have decided to try a different strategy. We send the minimum amount of data to DASHER to get back date jitters and item ids, and then the rest of the data gets put (anonymized) into Google Datastore.


### 3. Request
We now want to give our request to the API and get back a lookup for what the "anonymized" ids are. (eg, an MRN mapping to a SUID). We post one request to the DASHER API, with the unit of "study" as the item, meaning that the request looks like this:

```
{
  "identifiers": [
    {
      "id": "1111111",
      "id_source": "Stanford MRN",
      "id_timestamp": "1999-01-01T00:00:00Z",
      "items": [
        {
          "id": "2222222",
          "id_source": "DCM Accession #",
          "id_timestamp": "2000-01-01T12:12:00Z"
        }
      ]
    }
  ]
}
```

Note that the first section is pertaining to an entity, in this case a `PatientID` that corresponds to an MRN, and the "items" for that entity include the study, which corresponds to a particular Dicom accession Number. In this case, although the images carry unique identifiers, we don't send that level of granularity to the API.


```
from som.api.identifiers import Client
cli = Client(study='irlhs')
result = cli.anonymize(ids=request, study=study)
```

However, the API can only handle 1000 items per entity. To help with this, we have a task that handles parsing the request into batches, and then reassembling into one object:

```   
# in sendit.apps.main.tasks.get
results = batch_anonymize(ids=request,
                           study=study,
                           bid=batch.id)
```

The returned data would normally have `results` at the top level, but this function unwraps that into a list, where each item is an entity with all of its items:

```
# First entity in the list has the following fields
results[0].keys()
dict_keys(['id_source', 'custom_fields', 'jittered_timestamp', 'items', 'id', 'jitter', 'suid'])

# Second entity in the list has 1616 items (images)
len(results[0]['items'])
1616

# The first image for the second entity has the following fields
results[1]['items'][0].keys()    
dict_keys(['id_source', 'custom_fields', 'jittered_timestamp', 'id', 'jitter', 'suid'])
```

### 4. Save
The JSON response, the complete extracted identifiers, and the batch are saved to a `BatchIdentifiers` object along with a pointer to the `Batch`.

```
batch_ids = BatchIdentifiers.objects.create(batch=batch,
                                            response=results,
                                            ids=ids)

```

To review each field:
 
 - batch is the database object that holds a connection to all images under a single entity and accession number
 - respones is the minimal response from DASHER that we can use as a lookup to update the images and extracted data.
 - ids is the complete extracted data from every item, including many custom fields that weren't sent to DASHER.


We now move into the next step to update, and then replace identifiers.

to be passed on to the function `replace_identifiers`.



## Replace Identifiers
There are two steps to replacement. Remember that we are uploading two things:
 - images to Google Storage
 - metadata to Google Datastore

and thus replacement means:
 - replace and clean the header metadata in the images
 - clean the metadata to send to DataStore.

You can find these actions under the `replace_identifiers` task, also in [main/tasks](sendit/apps/main/update.py).


### 1. Prepare Identifiers
Before we replace anything in the data, we need to update our `ids` data structure with information from the SOM DASHER API response. Since this first operation is specific to the SOM and sendit application, we import from the `som` module:

```
from som.api.identifiers.dicom import prepare_identifiers
prepared = prepare_identifiers(response=batch_ids.response)
```

Remember that each of `batch_ids.response` is a single entity with some number of items nested under it. We need to unwrap it to return to this kind of organization:

```
prepared[entity][item] = {"field1": "value1", ... , "fieldN": "valueN"}
```

This format specific to the SOM API with entities and their metadata on a top level, and then a list of items, each with metadata, doesn't map nicely onto the organization of images. Each image needs the entity represented with its own metadata, and this is the format that the `deid` application is expecting. So we have a simple function to update the original identifiers with the response from the API:

```
from som.api.identifiers import update_identifiers
updated = update_identifiers(ids=batch_ids.ids,
                             updates=prepared)
batch_ids.ids = updated
batch_ids.save()
```

and note the last two lines, we replace our original ids with the updated structure.

Essentially the `ids` datastructure is returned and updated with our response from the som DASHER API. The reason that the functions `prepare_identifiers` and `update_identifiers` are separate from the general anonymization module, `deid`, is because `deid` is agnostic to the specific way that we want to update our identifiers. In our case, we are simply adding fields like `jitter` and `timestamp_jitter` to the data structure, for use during replacement.


### 2. Clean Identifiers
Let's review where we are at. The header metadata was originally extracted into the variable `ids` (saved under `batch_ids.ids`) and then updated with the content from a call to the DASHER API (again updated to `batch_ids.ids`) and now we want to give this data structure to a function in deid called `clean_identifiers`. This function is going to do the majority of work to do replacements, and apply a set of rules we have specified in our [deid.dicom](https://github.com/pydicom/deid/blob/master/deid/data/deid.dicom.blacklist). The clean function will also take into account the default set of values to keep, which are specified in the deid module's [config.json](https://github.com/vsoch/som/blob/master/som/api/identifiers/dicom/settings.py#L28). For the data sent to Google Cloud, since it doesn't make sense to send an empty field, the default action is `REMOVE`.

```
WARNING Field PresentationLUTShape is not present.
WARNING Field ContentLabel is not present.
WARNING Field ImageRotation is not present.
WARNING Field TimezoneOffsetFromUTC is not present.
WARNING 38 fields set for default action REMOVE
DEBUG StudyDate,SeriesInstanceUID,PatientName,PerformedProcedureStepStartDate,AcquisitionDate,AccessionNumber,RequestingService,
ContentDate,RequestAttributesSequence,StationName,jitter,SeriesTime,ReferringPhysicianName,PatientAddress,
item_timestamp,DistanceSourceToDetector,StudyTime,SeriesDate,Exposure,StudyInstanceUID,PatientAge,
NameOfPhysiciansReadingStudy,AdditionalPatientHistory,DistanceSourceToPatient,PerformingPhysicianName,
entity_id,InstitutionName,InstanceCreationTime,PerformedProcedureStepDescription,
FillerOrderNumberImagingServiceRequest,item_id,PerformedProcedureStepStartTime,
ContentTime,AcquisitionTime,entity_timestamp,SeriesNumber,StudyID,OperatorsName
```

This same set of operations and standard is done for the imaging data, but the default action is `BLANK` so all original headers are preserved. Sequences are removed by default. In addition, the images are renamed according to their assigned suid.

## Customizing anonymization
If you have a different use case, you have several options for customizing this step.

1. you can specify a different `config.json` to the get_identifiers function, in the case that you want a different set of rules applied to the orginal data extraction.
2. you can implement a new module (for example, for a different data type) by submitting a PR to the identifiers repository.
3. If you don't use DASHER, or do something entirely different, you have complete control to not use these som provided functions at all, in which case you will want to tweak the functions in the [tasks](../sendit/apps/main/tasks) folder.


At this point, we have finished the anonymization process (for header data, pixel anonymization is a separate thing still need to be developed) and can move on to [storage.md](storage.md)
