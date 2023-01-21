# Directus + Meilisearch

This is a starter to get Directus up and running with Meilisearch for searching.

Prerequisites:

- Docker and Docker Compose need to be installed - follow the instructions on the Docker website.

## 1. Editing env variables and starting up

Getting started:

- Edit a few things in `docker-compose.yml`:
  - directus.environment.KEY - generate random UUID
  - directus.environment.SECRET - generate random UUID
  - directus.environment.ADMIN_EMAIL - set it to what you want
  - directus.environment.ADMIN_PASSWORD - set it to what you want
  - directus.environment.MEILISEARCH_MASTER_KEY (needs to match meilisearch.environment.MEILI_MASTER_KEY)
  - meilisearch.environment.MEILI_MASTER_KEY (needs to match directus.environment.MEILISEARCH_MASTER_KEY)
- run `docker-compose pull` to pull the container images
- run `docker-compose up -d` to start the containers

## 2. Make sure you can access Directus and Meilisearch

Directus should be accessible at http://localhost:8055 in your web browser. You can log in with the admin details you set above.

Meilisearch should be accessible at http://localhost:7700 in your web browser. You can view the mini dashboard using the 'master key' you set above.

## 3. Create your collection in the Directus

- Log in at http://localhost:8055
- Go Settings -> Data Model -> Create a collection.
- As an example, you could:
  - Create a collection called 'documents'
  - Add a 'title' field - regular text input
  - Add a 'content' field - WYSIWYG content input

## 4. Create the flow in Directus

- In the Directus Settings, click on Flows
- Click the '+' to create a new flow
- Call it 'Send to Meilisearch'

### Trigger Setup

- Trigger Setup:
  - Type: 'Event Hook', 'Action (Non-Blocking)'
  - Scope: 'items.create, items.update'
- Collection - choose the collection you created above.

### Operation 1: run script to get the document ID

When the 'items.create' event hook runs, the ID is available as "$trigger.key".

When the 'items.update' event hook runs, the ID is available at "$trigger.keys[0]".

So we need to run a script that gets the ID from the payload, regardless of which hook runs.

Create an operation with the type 'Run Script' and enter this code:

```js
module.exports = async function (data) {
  // Find the ID in the payload

  if (data["$trigger"].key) return data["$trigger"].key;
  if (data["$trigger"].keys) return data["trigger"].keys[0];
};
```

### Operation 2: read data to lookup the document in the DB

Using the ID from the payload, we need to look up the data from the DB.

Create an operation with the type 'Read Data'.

Set 'collection' to your 'documents' collection.

Set 'IDs' to `{{$last}}`.

### Operation 3: make request to Meilisearch

At this stage, we have the document's info available in our JSON payload at `{{$last}}`.

We want to send that JSON payload to the Meilisearch API running at `http://meilisearch:7700`.

Create an operation with the type 'Webhook / Request URL'.

Set 'method' to POST. Set `URL` to `http://meilisearch:7700/indexes/documents/documents`. If you want a different 'ID' for the index, you can set it differently. For example if you want the index to be called webpages, the URL would be `http://meilisearch:7700/indexes/webpages/documents`.

Add a couple of headers: - `Content-Type`: `application/json` - `Authorization`: `Bearer {{$env.MEILISEARCH_MASTER_KEY}}`

The `MEILISEARCH_MASTER_KEY` is available in `$env` because we added it to the `FLOWS_ENV_ALLOW_LIST` in our `docker-compose.yml` file.

In 'request body', set it to `{{$last}}`.

## 5. Test it out

Once you've saved your flow, you can test it out. Create a new record in Directus.

You can go back to Settings -> Flows, click on the flow and then click 'Logs' in the right hand sidebar to check that each operation is running properly.

You can also check the Meilisearch mini-dashboard at `http://localhost:7700` to check that records are being added.

## Where to from here?

I'm hoping to flesh this out into a more detailed blog post or video resource.

From here you could try setting up a front-end so that users can search your content. You could check out the [Meilisearch docs](http://meilisearch:7700/indexes/documents/documents) for more info there.
