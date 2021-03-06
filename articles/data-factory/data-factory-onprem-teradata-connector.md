﻿---
title: Move data from Teradata | Microsoft Docs
description: Learn about Teradata Connector for the Data Factory service that lets you move data from Teradata Database
services: data-factory
documentationcenter: ''
author: linda33wj
manager: jhubbard
editor: monicar

ms.assetid: 98eb76d8-5f3d-4667-b76e-e59ed3eea3ae
ms.service: data-factory
ms.workload: data-services
ms.tgt_pltfrm: na
ms.devlang: na
ms.topic: article
ms.date: 11/01/2016
ms.author: jingwang

---
# Move data from Teradata using Azure Data Factory
This article outlines how you can use the Copy Activity in an Azure data factory to move data to from Teradata to another data store. This article builds on the [data movement activities](data-factory-data-movement-activities.md) article, which presents a general overview of data movement with copy activity and supported data store combinations.

Data factory supports connecting to on-premises Teradata sources via the Data Management Gateway. See [moving data between on-premises locations and cloud](data-factory-move-data-between-onprem-and-cloud.md) article to learn about Data Management Gateway and step-by-step instructions on setting up the gateway.

> [!NOTE]
> Gateway is required even if the Teradata is hosted in an Azure IaaS VM. You can install the gateway on the same IaaS VM as the data store or on a different VM as long as the gateway can connect to the database.
>
>

Data factory supports only moving data from Teradata to other data stores, not from other data stores to Teradata.

## Supported versions and installation
For Data Management Gateway to connect to the Teradata Database, you need to install the [.NET Data Provider for Teradata](http://go.microsoft.com/fwlink/?LinkId=278886) version 14 or above on the same system as the Data Management Gateway. Teradata version 12 and above is supported.

> [!NOTE]
> See [Troubleshoot gateway issues](data-factory-data-management-gateway.md#troubleshooting-gateway-issues) for tips on troubleshooting connection/gateway related issues.
>
>

## Copy data wizard
The easiest way to create a pipeline that copies data from Teradata to any of the supported sink data stores is to use the Copy data wizard. See [Tutorial: Create a pipeline using Copy Wizard](data-factory-copy-data-wizard-tutorial.md) for a quick walkthrough on creating a pipeline using the Copy data wizard.

The following example provides sample JSON definitions that you can use to create a pipeline by using [Azure portal](data-factory-copy-activity-tutorial-using-azure-portal.md) or [Visual Studio](data-factory-copy-activity-tutorial-using-visual-studio.md) or [Azure PowerShell](data-factory-copy-activity-tutorial-using-powershell.md). They show how to copy data from Teradata to Azure Blob Storage. However, data can be copied to any of the sinks stated [here](data-factory-data-movement-activities.md#supported-data-stores-and-formats) using the Copy Activity in Azure Data Factory.   

### Sample: Copy data from Teradata to Azure Blob
This sample shows how to copy data from a Teradata database to an Azure Blob Storage. However, data can be copied **directly** to any of the sinks stated [here](data-factory-data-movement-activities.md#supported-data-stores-and-formats) using the Copy Activity in Azure Data Factory.  

The sample has the following data factory entities:

1. A linked service of type [OnPremisesTeradata](data-factory-onprem-teradata-connector.md#teradata-linked-service-properties).
2. A linked service of type [AzureStorage](data-factory-azure-blob-connector.md#azure-storage-linked-service).
3. An input [dataset](data-factory-create-datasets.md) of type [RelationalTable](data-factory-onprem-teradata-connector.md#teradata-dataset-type-properties).
4. An output [dataset](data-factory-create-datasets.md) of type [AzureBlob](data-factory-azure-blob-connector.md#azure-blob-dataset-type-properties).
5. The [pipeline](data-factory-create-pipelines.md) with Copy Activity that uses [RelationalSource](data-factory-onprem-teradata-connector.md#teradata-copy-activity-type-properties) and [BlobSink](data-factory-azure-blob-connector.md#azure-blob-copy-activity-type-properties).

The sample copies data from a query result in Teradata database to a blob every hour. The JSON properties used in these samples are described in sections following the samples.

As a first step, setup the data management gateway. The instructions are in the [moving data between on-premises locations and cloud](data-factory-move-data-between-onprem-and-cloud.md) article.

**Teradata linked service:**

    {
        "name": "OnPremTeradataLinkedService",
        "properties": {
            "type": "OnPremisesTeradata",
            "typeProperties": {
                "server": "<server>",
                "authenticationType": "<authentication type>",
                "username": "<username>",
                "password": "<password>",
                "gatewayName": "<gatewayName>"
            }
        }
    }

**Azure Blob storage linked service:**

    {
        "name": "AzureStorageLinkedService",
        "properties": {
            "type": "AzureStorageLinkedService",
            "typeProperties": {
                "connectionString": "DefaultEndpointsProtocol=https;AccountName=<AccountName>;AccountKey=<AccountKey>"
            }
        }
    }


**Teradata input dataset:**

The sample assumes you have created a table “MyTable” in Teradata and it contains a column called “timestamp” for time series data.

Setting “external”: true informs the Data Factory service that the table is external to the data factory and is not produced by an activity in the data factory.

    {
        "name": "TeradataDataSet",
        "properties": {
            "published": false,
            "type": "RelationalTable",
            "linkedServiceName": "OnPremTeradataLinkedService",
            "typeProperties": {
            },
            "availability": {
                "frequency": "Hour",
                "interval": 1
            },
            "external": true,
            "policy": {
                "externalData": {
                    "retryInterval": "00:01:00",
                    "retryTimeout": "00:10:00",
                    "maximumRetry": 3
                }
            }
        }
    }


**Azure Blob output dataset:**

Data is written to a new blob every hour (frequency: hour, interval: 1). The folder path for the blob is dynamically evaluated based on the start time of the slice that is being processed. The folder path uses year, month, day, and hours parts of the start time.

    {
        "name": "AzureBlobTeradataDataSet",
        "properties": {
            "published": false,
            "location": {
                "type": "AzureBlobLocation",
                "folderPath": "mycontainer/teradata/yearno={Year}/monthno={Month}/dayno={Day}/hourno={Hour}",
                "format": {
                    "type": "TextFormat",
                    "rowDelimiter": "\n",
                    "columnDelimiter": "\t"
                },
                "partitionedBy": [
                    {
                        "name": "Year",
                        "value": {
                            "type": "DateTime",
                            "date": "SliceStart",
                            "format": "yyyy"
                        }
                    },
                    {
                        "name": "Month",
                        "value": {
                            "type": "DateTime",
                            "date": "SliceStart",
                            "format": "MM"
                        }
                    },
                    {
                        "name": "Day",
                        "value": {
                            "type": "DateTime",
                            "date": "SliceStart",
                            "format": "dd"
                        }
                    },
                    {
                        "name": "Hour",
                        "value": {
                            "type": "DateTime",
                            "date": "SliceStart",
                            "format": "HH"
                        }
                    }
                ],
                "linkedServiceName": "AzureStorageLinkedService"
            },
            "availability": {
                "frequency": "Hour",
                "interval": 1
            }
        }
    }


**Pipeline with Copy activity:**

The pipeline contains a Copy Activity that is configured to use the input and output datasets and is scheduled to run hourly. In the pipeline JSON definition, the **source** type is set to **RelationalSource** and **sink** type is set to **BlobSink**. The SQL query specified for the **query** property selects the data in the past hour to copy.

    {
        "name": "CopyTeradataToBlob",
        "properties": {
            "description": "pipeline for copy activity",
            "activities": [
                {
                    "type": "Copy",
                    "typeProperties": {
                        "source": {
                            "type": "RelationalSource",
                            "query": "$$Text.Format('select * from MyTable where timestamp >= \\'{0:yyyy-MM-ddTHH:mm:ss}\\' AND timestamp < \\'{1:yyyy-MM-ddTHH:mm:ss}\\'', SliceStart, SliceEnd)"
                        },
                        "sink": {
                            "type": "BlobSink",
                            "writeBatchSize": 0,
                            "writeBatchTimeout": "00:00:00"
                        }
                    },
                    "inputs": [
                        {
                            "name": "TeradataDataSet"
                        }
                    ],
                    "outputs": [
                        {
                            "name": "AzureBlobTeradataDataSet"
                        }
                    ],                    
                    "policy": {
                        "timeout": "01:00:00",
                        "concurrency": 1
                    },
                    "scheduler": {
                        "frequency": "Hour",
                        "interval": 1
                    },
                    "name": "TeradataToBlob"
                }
            ],
            "start": "2014-06-01T18:00:00Z",
            "end": "2014-06-01T19:00:00Z",
            "isPaused": false
        }
    }


## Teradata linked service properties
The following table provides description for JSON elements specific to Teradata linked service.

| Property | Description | Required |
| --- | --- | --- |
| type |The type property must be set to: **OnPremisesTeradata** |Yes |
| server |Name of the Teradata server. |Yes |
| authenticationType |Type of authentication used to connect to the Teradata database. Possible values are: Anonymous, Basic, and Windows. |Yes |
| username |Specify user name if you are using Basic or Windows authentication. |No |
| password |Specify password for the user account you specified for the username. |No |
| gatewayName |Name of the gateway that the Data Factory service should use to connect to the on-premises Teradata database. |Yes |

See [Move data between on-premises sources and the cloud with Data Management Gateway](data-factory-move-data-between-onprem-and-cloud.md) for details about setting credentials for an on-premises Teradata data source.

## Teradata dataset type properties
For a full list of sections & properties available for defining datasets, see the [Creating datasets](data-factory-create-datasets.md) article. Sections such as structure, availability, and policy of a dataset JSON are similar for all dataset types (Azure SQL, Azure blob, Azure table, etc.).

The **typeProperties** section is different for each type of dataset and provides information about the location of the data in the data store. Currently, there are no type properties supported for the Teradata dataset.

## Teradata copy activity type properties
For a full list of sections & properties available for defining activities, see the [Creating Pipelines](data-factory-create-pipelines.md) article. Properties such as name, description, input and output tables, and policies are available for all types of activities.

Properties available in the typeProperties section of the activity on the other hand vary with each activity type. For Copy activity, they vary depending on the types of sources and sinks.

When source is of type **RelationalSource** (which includes Teradata) the following properties are available in **typeProperties** section:

| Property | Description | Allowed values | Required |
| --- | --- | --- | --- |
| query |Use the custom query to read data. |SQL query string. For example: select * from MyTable. |Yes |

[!INCLUDE [data-factory-structure-for-rectangualr-datasets](../../includes/data-factory-structure-for-rectangualr-datasets.md)]

## Type mapping for Teradata
As mentioned in the [data movement activities](data-factory-data-movement-activities.md) article, the Copy activity performs automatic type conversions from source types to sink types with the following 2-step approach:

1. Convert from native source types to .NET type
2. Convert from .NET type to native sink type

When moving data to Teradata, the following mappings are used from Teradata type to .NET type.

| Teradata Database type | .NET Framework type |
| --- | --- |
| Char |String |
| Clob |String |
| Graphic |String |
| VarChar |String |
| VarGraphic |String |
| Blob |Byte[] |
| Byte |Byte[] |
| VarByte |Byte[] |
| BigInt |Int64 |
| ByteInt |Int16 |
| Decimal |Decimal |
| Double |Double |
| Integer |Int32 |
| Number |Double |
| SmallInt |Int16 |
| Date |DateTime |
| Time |TimeSpan |
| Time With Time Zone |String |
| Timestamp |DateTime |
| Timestamp With Time Zone |DateTimeOffset |
| Interval Day |TimeSpan |
| Interval Day To Hour |TimeSpan |
| Interval Day To Minute |TimeSpan |
| Interval Day To Second |TimeSpan |
| Interval Hour |TimeSpan |
| Interval Hour To Minute |TimeSpan |
| Interval Hour To Second |TimeSpan |
| Interval Minute |TimeSpan |
| Interval Minute To Second |TimeSpan |
| Interval Second |TimeSpan |
| Interval Year |String |
| Interval Year To Month |String |
| Interval Month |String |
| Period(Date) |String |
| Period(Time) |String |
| Period(Time With Time Zone) |String |
| Period(Timestamp) |String |
| Period(Timestamp With Time Zone) |String |
| Xml |String |

[!INCLUDE [data-factory-column-mapping](../../includes/data-factory-column-mapping.md)]

[!INCLUDE [data-factory-type-repeatability-for-relational-sources](../../includes/data-factory-type-repeatability-for-relational-sources.md)]

## Performance and Tuning
See [Copy Activity Performance & Tuning Guide](data-factory-copy-activity-performance.md) to learn about key factors that impact performance of data movement (Copy Activity) in Azure Data Factory and various ways to optimize it.
