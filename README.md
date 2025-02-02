# Movie Catalog Manager With Azure Function.

This project aims to create an movie catalog manager with Azure Function and CosmosDB. We'll show step by step of this process.

## ⚙️ Steps:


These are the steps we will follow to structure this video catalog manager. Next, we will break down each of these steps in detail.


- Create the Cloud Infrastructure;

- Create an Azure Function to save files to the storage account;

- Create an Azure Function to save to CosmosDB;

- Create an Azure Function to filter records in CosmosDB;

- Create an Azure Function to list records in CosmosDB.


### Create the Cloud Infrastructure:


The first Microsoft Azure role we will create is an API Manager. After creating this manager, we need to create the CosmosDB NoSQL database and a Storage Account.

Finally, we need to create a Front End for the application. In this step, we will use Streamlit.io just to have some visualization.

These creation steps are simple, just follow what Microsoft Azure asks for. So we won't go into details at this stage. I believe a relevant topic to understand here is the format in which records are made in CosmosDB.

```json

{
  "title" : "<Movie Title>",
  "video" : "<Video Path>/*.mp4",
  "thub" : "<Template Path>/*.png",
  "synopsis" : "<Movie Synopsis>",
  "genre" : "<Movie genre>"
}

```


### Create an Azure Function to save files to the storage account:


### Create an Azure Function to save to CosmosDB:


### Create an Azure Function to filter records in CosmosDB:


### Create an Azure Function to list records in CosmosDB:

