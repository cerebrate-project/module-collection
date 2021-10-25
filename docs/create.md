# Module development

This document is meant as an introduction to the module development for Cerebrate

## Scope

Generally modules are meant to accomplish a similar set of tasks:

- Manage Brood to Brood interconnections (e.g. I want to connect my MISP to your MISP)
- Local fleet management (e.g. I want to manage my local MISP instance's configuration)
- Diagnostics and statistics (I want to gather usage and health monitoring data in one place)
- Sharing rule / contact information management (e.g. I want to inform my MISP of a sharing group or a set of organisations)


## Getting started

There is a [basic skeleton](examples/SkeletonConnector.php) that you can use as a basic starting point.

Basically the bare minimum for a module to be considered "complete" and loadable, it has to follow the following skeleton:

```
<?php
namespace MyToolConnector;

require_once(ROOT . '/src/Lib/default/local_tool_connectors/CommonConnectorTools.php');
use CommonConnectorTools\CommonConnectorTools;

class MyToolConnector extends CommonConnectorTools
{
    public $description = 'A connector for MyTool.';
    public $connectorName = 'MyToolConnector';
    public $name = 'MyTool';
    public $version = '1';

    public $exposedFunctions = [];

    public function health(Object $connection): array
    {
        return 0;
    }
}
```

### Additional imports

Refer to the CakePHP documentation for additional available libraries of the framework that you may wish to use, though as a bare minimum, it is highly recommended to also import HTTP client libraries as well as error handling, for example:

```
use Cake\Http\Client;
use Cake\Http\Exception\NotFoundException;
use Cake\Http\Exception\MethodNotAllowedException;
```

### The mandatory health() function

This is a simplistic function that is required by all modules to be implemented, and it is meant to simply return a quick check on whether the connected tool is configured correctly and usable. The function returns a numeric status code depending on the state:

- 0: UNKNOWN
- 1: OK
- 2: ISSUES
- 3: ERROR

### Adding functionalities to our 

Additional functionalities need to be implemented with one public function representing each functionality. These also need to be mapped via the exposedFunctions class.

Exposed functions can always be accessed via `/localTools/action/[local_tool_connection_id]/[action_name]/optional_parameter1/optional_parameter2`

#### Action types

Generally we differentiate between the following functionality types:

- index: A list of data gathered from the tool, optionally with actions associated to each row
- formAction: An action, which when triggered opens a modal with a form. These actions normally have two main functionalities: Display a form (GET) and process submitted data (POST). The latter is also API exposed.


#### Action structure

When creating module actions, we have access to all underlying functonalities that the Cerebrate code-base internally offers, including reading and writing data to any Cerebrate facilities via the appropriate model actions. Make sure to limit the local data access to whatever is absolutely necessary (such as retrieving sharing groups from Cerebrate when feeding a connected local tool with them).

Action functions are generally split into two main parts:

- The actual execution of the request
- Building a response for the UI / API

The first is done by interacting with Cerebrate data as well as querying the local tool (via HTTP Client for example) and processing the response.

Building a response for the UI happens by feeding the parametrised view builders of Cerebrate, which will automatically build indexes, forms, modals based on Cerebrate's usual look and feel.

If you would like to avoid large, complex functions, feel free to populate your module class with private functions that assist you in your exposed action functions' workflows, as well as extracting common functionalities shared by multiple actions (such as handlers for the communications with local tools).

An example for such an extracted private function (the function handling all GET requests in the MISP module):

```
    private function getData(string $url, array $params): Response
    {
        if (empty($params['connection'])) {
            throw new NotFoundException(__('No connection object received.'));
        }
        if (!empty($params['sort'])) {
            $list = explode('.', $params['sort']);
            $params['sort'] = end($list);
        }
        if (!isset($params['limit'])) {
            $params['limit'] = 50;
        }
        $url = $this->urlAppendParams($url, $params);
        $response = $this->HTTPClientGET($url, $params['connection']);
        if ($response->isOk()) {
            return $response;
        } else {
            if (!empty($params['softError'])) {
                return $response;
            }
            throw new NotFoundException(__('Could not retrieve the requested resource.'));
        }
    }
```

#### Common functionalities

We by default inherit a set of functionalities we can reuse via CommonConnectorTools.php. This is the class that all Cerebrate modules extend and contain the code to handle a set of basic tasks such as:

- Executing actions
- Checking the tool's status
- Capturing ingested contact database items retrieved from a local tool (organisations, individuals, sharing groups)

## Building responses

Reponses to the UI are always returned as an array of parameters, with the actual data being contained within the array itself. An example for an index function's representation:

### Index actions

```
[
    'type' => 'index',
    'data' => [
        'data' => $data,
        'skip_pagination' => 1,
        'top_bar' => [
            'children' => []
        ],
        'fields' => [
            [
                'name' => __('My first field'),
                'data_path' => 'field1',
            ],
            [
                'name' => __('My second field'),
                'data_path' => 'foo.field2',
            ],
        ],
        'title' => __(MyTool's data index),
        'description' => false,
        'pull' => 'right',
        'actions' => []
    ]
];
```

The above would generate an index, with each element in $data being a row in the tabe. Each row would have 2 columns displayed, `$data[$row_id]['field1']` and `$data[$row_id]['foo']['field2']`.

#### Defining index global functionalities

If we wanted to add additional functionalities to the index, such as toggles, filters, we could pass those via the `top_bar` key. A common use-case for this is adding a quick search bar, such as the below:

```
'top_bar' => [
    'children' => [
        [
            'type' => 'search',
            'button' => __('Filter'),
            'placeholder' => __('Enter value to search'),
            'data' => '',
            'searchKey' => 'value'
        ]
    ]
],
```

This will add a filter text entry on top of our index, which can be accessed in your action by using the passed $params array. The above filter would populate $params['value'] for example (defined by the searchKey).

#### Refining index fields

Additionally, the individual fields in the index can be further refined, rather than just offering text representations of the referenced data.

```
[
'name' => 'Criticality',
'sort' => 'level',
'data_path' => 'level',
'arrayData' => [
    0 => 'Critical',
    1 => 'Recommended',
    2 => 'Optional'
],
'element' => 'array_lookup_field'
],
```

The above example shows an enum field with a mapped representation using the "array\_lookup\_field" element. For a comprehensive list of elements, refer to your Cerebrate's `/var/www/Cerebrate/templates/genericElements/IndexTable/Fields/` directory.

If you set the `sort` key, then users can sort the index by clicking a column's name. You would need to handle that in your action if you wish for sorting to be available. You can retrieve the selected sorting rules via the passed param array ($params['sort'])

### Form actions

When handling dialogues in the module system, functions have generally two use-cases. GET requests will fetch a form and POST actions will execute the requested action with the passed parameters. To handle both these use-cases, you can simply inspect the request http method such as this:

```
if ($params['request']->is(['get'])) {
    // generate form for the user
} elseif ($params['request']->is(['post'])) {
    // handle the posted data
}
```

#### GET requests

To generate a form, we use the form factories of cerebrate by parametrising the resuting modal:

```
[
	'data' => [
	    'title' => __('Fetch organisation'),
	    'description' => __('Fetch and create/update organisation ({0}) from MISP.', $params['uuid']),
	    'submit' => [
		'action' => $params['request']->getParam('action')
	    ],
	    'url' => ['controller' => 'localTools', 'action' => 'action', $params['connection']['id'], 'fetchOrganisationAction', $params['uuid']]
]
```

The above is an example from the MISP module's fetchOrganisationAction()'s GET code branch. It will generate a simple confirmation modal.

The url key is used to point the action to the destination of where the contents should be posted. Keep in mind that whilst here we have an example of an action that has a simple GET and POST code branch, you could chain multiple actions and create longer dialogue sequences by POSTing the contents to a different endpoint. 

The above example does not allow for any user set details as it's only a confirmation modal, but you can turn an action into an interactive form by simply adding form fields to it, using the `fields` key nested in the `data` key:

```
'fields' => [
        [
            'field' => 'connection_ids',
            'type' => 'hidden',
            'value' => $params['connection_ids']
        ],
        [
            'field' => 'method',
            'label' => __('Method'),
            'type' => 'dropdown',
            'options' => ['GET' => 'GET', 'POST' => 'POST']
        ],
	[
	    'field' => 'url',
	    'label' => __('Relative URL'),
	    'type' => 'text',
	]
]
```

The above will create two form fields, a hidden conntetion_ids parameter passed along from the function and a dropdown for the user to set and a text entry field. For the available field elements, have a look at your Form factory's field directory at `/var/www/cerebrate/templates/element/IndexTable/Fields`


#### POST requests

For post requests, we most likely will want to comminucate the user request to the local tool, via whichever means we would talk to the tool. The most commonly used way of interacting with a tool would be via the HTTP client.

Keep in mind, that all configuration and tool connection specific data is contained in the $params - so you can retrieve stored data such as the local tool's URL, auth key, etc from there, provided they were correctly configured when encoding the connection.

For the actual response, we generally return a JSON that contains the request's outcome as well as any data returned in the following format:

```
[
    "success": 1,
    "message": "Action completed",
    "data": ""
]
```


