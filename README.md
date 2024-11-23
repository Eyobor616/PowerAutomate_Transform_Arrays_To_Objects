# PowerAutomate_Transform_Arrays_To_Objects

This flow receives the JSON data of Groups using the Office365Groups Connector
The Flow returns an Object {}, but what is needed to  be extracted is the value: [] which is an array of object 
{
  "@odata.context": "https://graph.microsoft.com/v1.0/$metadata#groups",
  "@odata.nextLink": "https://graph.microsoft.com/v1.0/groups?$skiptoken=xxx-xx-x-x-x--x-x-x--xx-xxxxxxxx",
  "value": [
    {
      "id": "xxx-xx-x-xxx-x-",
      "deletedDateTime": null,
      "classification": null,
      "createdDateTime": "2024-10-30T12:30:57Z",
      "creationOptions": []......

#TURN ARRAY INTO OBJECTS (Take out the surrounding [])

Use the join() expression targetted at the value (  @{join(body('GetGroupConn')?['value'],',')}  ) to take out the surrounding [] from the returned json.
THE RESULT
    {
      "id": "xxx-xx-x-xxx-x-",
      "deletedDateTime": null,
      "classification": null
      ...
    }
The reason for doing this is that the 'Append to Array variable accepts values with surrounding {}

#TURN OBJECTS TO ARRAY (Surround with [])
@{json(concat('[', outputs('Compose'), ']'))
}

You need to do this in order to using in a Select statement or parse to PowerApps


#PowerApps Code for turning JSON into Collection

Set(varGetGroups, GetM365Groups.Run().resp); //Run Flow

If(!IsBlank(varGetGroups),

    ClearCollect(
    colDistributionList_Group,
            IsBlank(groupTypes), mailEnabled,  !securityEnabled,
            ForAll(Table(ParseJSON(varGetGroups)),  
            {
                displayName: Text(ThisRecord.Value.displayName),
                id: Text(ThisRecord.Value.id),
                description: Text(ThisRecord.Value.description),
                groupTypes: Text(First(ThisRecord.Value.groupTypes)),
                mail: Text(ThisRecord.Value.mail),
                mailEnabled: Text(ThisRecord.Value.mailEnabled),
                onPremisesDomainName: Text(ThisRecord.Value.onPremisesDomainName),
                securityEnabled: Text(ThisRecord.Value.securityEnabled)
            }
        ));

);


The Filter Operation filter for Distribution list only


#FULL CODE PREVIEW


1. Initialize an Empty Array
2. Get Groups (Office 365 Groups) - URL - https://graph.microsoft.com/v1.0/groups, GET
3. Append to array variable-
     Name: GroupArray
     Value:@{join(body('GetGroupConn')?['value'],',')}
4. Initialize variable-NextLinkAvailable // to store the next available link.. This returns the next link to load the next batch of Groups/items.. 100/batch
     Name: varNextLink
     Type: String
     "value": "@body('GetGroupConn')?['@odata.nextLink']"


   #DO UNTIL LOOP
   {
  "type": "Until",
  "expression": "@equals(variables('varNextLink'),'')",
  "limit": {
    "timeout": "PT1H"
  },
  "actions": {
    "Append_to_array_variable-2": {
      "type": "AppendToArrayVariable",
      "inputs": {
        "name": "GroupArray",
        "value": "@join(body('GetGroupConn_2')?['value'],',')"
      },
      "runAfter": {
        "GetGroupConn_2": [
          "Succeeded"
        ]
      },
      "metadata": {
        "operationMetadataId": "6899eef5-b499-4f41-99c1-4a06c75cf6cd"
      }
    },
    "Set_variable": {
      "type": "SetVariable",
      "inputs": {
        "name": "varNextLink",
        "value": "@body('GetGroupConn_2')?['@odata.nextLink']"
      },
      "runAfter": {
        "Append_to_array_variable-2": [
          "Succeeded"
        ]
      },
      "metadata": {
        "operationMetadataId": "f13b5224-45aa-4688-98cf-b3253d1d7e6d"
      }
    },
    "GetGroupConn_2": {
      "type": "OpenApiConnection",
      "inputs": {
        "parameters": {
          "Uri": "@variables('varNextLink')",
          "Method": "GET",
          "ContentType": "application/json"
        },
        "host": {
          "apiId": "/providers/Microsoft.PowerApps/apis/shared_office365groups",
          "connection": "shared_office365groups",
          "operationId": "HttpRequestV2"
        }
      },
      "metadata": {
        "operationMetadataId": "941e252f-10d8-4aa0-a969-766340fef9e3"
      }
    }
  },
  "runAfter": {
    "Initialize_variable-NextLinkAvailable": [
      "Succeeded"
    ]
  },
  "metadata": {
    "operationMetadataId": "a2d01241-2372-45bf-8172-36ff80f3d5ec"
  }
}


#ACTIONS AFTER THE DO UNTIL LOOP

COMPOSE //
{
  "type": "Compose",
  "inputs": "@join(variables('GroupArray'),',')",
  "runAfter": {
    "Do_until": [
      "Succeeded"
    ]
  }
}


![image](https://github.com/user-attachments/assets/bb95ec5b-fd99-4f72-af43-7700c594d4fb)



