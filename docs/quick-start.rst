===========
Quick Start
===========

Creating a client
-----------------

You can quickly get up and running by using a web service client's factory method to instantiate clients as needed.

.. code-block:: php

    <?php

    // Include the SDK using the Composer autoloader
    require 'vendor/autoload.php';

    use Aws\S3\S3Client;
    use Aws\Common\Enum\Region;

    // Instantiate the S3 client with your AWS credentials and desired AWS region
    $client = S3Client::factory(array(
        'key'    => 'your-aws-access-key-id',
        'secret' => 'your-aws-secret-access-key',
    ));

**Note:** Instantiating a client without providing credentials causes the client to attempt to retrieve `IAM Instance
Profile credentials
<http://docs.amazonwebservices.com/AWSEC2/latest/UserGuide/UsingIAM.html#UsingIAMrolesWithAmazonEC2Instances>`_.

Commands
--------

You can then invoke service operations on the client by calling the operation name and providing an associative array
of parameters. Service operation methods like Amazon S3's ``createBucket()`` don't actually exist on a client. These
methods are implemented using the ``__call()`` magic method of a client. These magic methods are derived from a Guzzle
`service description <http://guzzlephp.org/guide/service/service_descriptions.html>`_ present in the
Resources/client.php file of each client. You can use the
`API documentation <http://docs.amazonwebservices.com/aws-sdk-php-2/latest/>`_ or directly view the service description
for help on what operations are available, what parameters are used for an operation, what values are provided in a
response model, and what exceptions are thrown by calling the operation.

.. code-block:: php

    $bucket = 'mybucket';

    $result = $client->createBucket(array(
        'Bucket' => $bucket
    ));

    // Wait until the bucket is created
    $client->waitUntil('BucketExists', array('Bucket' => $bucket));

Executing commands
~~~~~~~~~~~~~~~~~~

Commands can be created in two ways: using magic methods (as shown above) or using the ``getCommand()`` method of a
client object.

.. code-block:: php

    // The magic method syntax via __call
    $result = $client->createBucket(array(/* ... */));

    // The "full" syntax
    $command = $client->getCommand('CreateBucket', array(/* ... */));
    $result = $command->getResult();

Using the "full" syntax, the return value is a ``Command`` object, which encapsulates the request and response of the
HTTP request to AWS. From the ``Command`` object, you can call the ``getResult()`` method or the ``execute()`` method to
get the parsed result. Additionally, you can call the ``getRequest()`` and ``getResponse()`` methods to get information
about the request and response, respectively (e.g., the status code or the raw response, headers sent in the request,
etc.).

The ``Command`` object supports a chainable syntax and can also be useful when you want to manipulate the request before
execution.

.. code-block:: php

    $result = $client->getCommand('ListObjects')
        ->set('MaxKeys', 50)
        ->set('Prefix', 'foo/baz/')
        ->getResult();

It also allows for executing multiple commands in parallel.

.. code-block:: php

    $ops = array();
    $ops[] = $client->getCommand('GetObject', array('Bucket' => 'foo', 'Key' => 'Bar'));
    $ops[] = $client->getCommand('GetObject', array('Bucket' => 'foo', 'Key' => 'Baz'));
    $client->execute($ops);

Response models
~~~~~~~~~~~~~~~

The result of executing a command will always return a ``Guzzle\Service\Resource\Model`` response model object. This
model can be used like an array and contains information about the JSON-schema structure of the model. Response models
are populated by parsing an HTTP response and pulling values out of a response based on rules found in the service
description of a client. You can use the API documentation of the SDK or directly reference the service description for
a list of data available in the response model of an operation.

.. code-block:: php

    $result = $client->getObject(array(
        'Bucket' => 'mybucket',
        'Key'    => 'test.txt'
    ));

    echo get_class($result);
    // Guzzle\Service\Resource\Model

    var_export($result->getKeys());
    // array('Body', 'DeleteMarker', 'Expiration', 'ContentLength', etc...)

    echo $result['ContentLength']);
    // 6

    echo $result['Body'];
    // hello!

    echo $result->getPath('Metadata/CustomValue');
    // Testing123

    var_export($result->getPath('Metadata/DoesNotExist'));
    // NULL

Using the service builder
-------------------------

When using the SDK, you have the option to use individual factory methods for each client or the ``Aws\Common\Aws``
class to build your clients. The ``Aws\Common\Aws`` class is a service builder and dependency injection container for
the SDK and is the recommended way for instantiating clients. The service builder allows you to share configuration
options between multiple services and pre-wires short service names with the appropriate client class.

The following example shows how to use the service builder to retrieve a ``Aws\DynamoDb\DynamoDbClient`` and perform the
``GetItem`` operation using the command syntax.

Passing an associative array of parameters as the first or second argument of ``Aws\Common\Aws::factory()`` treats the
parameters as shared across all clients generated by the builder. In the example, we tell the service builder to use the
same credentials for every client.

.. code-block:: php

    <?php

    require 'vendor/autoload.php';

    use Aws\Common\Aws;
    use Aws\Common\Enum\Region;
    use Aws\DynamoDb\Exception\DynamoDbException;

    // Create a service building using shared credentials for each service
    $aws = Aws::factory(array(
        'key'    => 'your-aws-access-key-id',
        'secret' => 'your-aws-secret-access-key',
        'region' => Region::US_WEST_2
    ));

    // Retrieve the DynamoDB client by its short name from the service builder
    $client = $aws->get('dynamodb');

    // Get an item from the "posts"
    try {
        $result = $client->getItem(array(
            'TableName' => 'posts',
            'Key' => $client->formatAttributes(array(
                'HashKeyElement' => 'using-dynamodb-with-the-php-sdk'
            )),
            'ConsistentRead' => true
        ));

        print_r($result['Item']);
    } catch (DynamoDbException $e) {
        echo 'The item could not be retrieved.';
    }

Passing an associative array of parameters to the first or second argument of ``Aws\Common\Aws::factory()`` will treat
the parameters as shared parameters across all clients generated by the builder. In the above example, we are telling
the service builder to use the same credentials for every client.

Error handling
--------------

An exception is thrown when an error is encountered. Be sure to use try/catch blocks when implementing error handling
logic in your applications. The SDK throws service specific exceptions when a server-side error occurs.

.. code-block:: php

    use Aws\Common\Aws;
    use Aws\S3\Exception\BucketAlreadyExistsException;

    $aws = Aws::factory('/path/to/my_config.json');
    $s3 = $aws->get('s3');

    try {
        $s3->createBucket(array('Bucket' => 'mybucket'));
    } catch (BucketAlreadyExistsException $e) {
        echo 'That bucket already exists! ' . $e->getMessage() . "\n";
    }

The HTTP response to the ``createBucket()`` method will receive a ``409 Conflict`` response with a
``BucketAlreadyExists`` error code. When the SDK sees the error code it will attempt to throw a named exception that
matches the name of the HTTP response error code. You can see a full list of supported exceptions for each client by
looking in the Exception/ directory of a client namespace. For example, src/Aws/S3/Exception contains many different
exception classes::

    .
    ├── AccessDeniedException.php
    ├── AccountProblemException.php
    ├── AmbiguousGrantByEmailAddressException.php
    ├── BadDigestException.php
    ├── BucketAlreadyExistsException.php
    ├── BucketAlreadyOwnedByYouException.php
    ├── BucketNotEmptyException.php
    [...]

Waiters
-------

One of the high-level abstractions provided by the SDK is the concept of "waiters". Waiters help make it easier to work
with eventually consistent systems by providing an easy way to wait on a resource to enter into a particular state by
polling the resource. You can find a list of the iterators supported by a client by viewing the docblock of a client.
Any ``@method`` tag that starts with "waitUntil" will utilize a waiter.

.. code-block:: php

    $client->waitUntil('BucketExists', array('Bucket' => 'mybucket'));

The above method invocation will instantiate a waiter and poll the bucket until it exists. If the waiter has to poll
the bucket too many times, it will throw an ``Aws\Common\Exception\RuntimeException`` exception.

You can tune the number of polling attempts issued by a waiter or the number of seconds to delay between each poll by
passing optional values prefixed with "waiter.":

.. code-block:: php

    $client->waitUntil('BucketExists', array(
        'Bucket ' => 'mybucket',
        'waiter.interval'     => 10.5,
        'waiter.max_attempts' => 3
    ));

Iterators
---------

Some AWS operations will return a paginated result set that requires subsequent requests in order to retrieve an entire
result. The AWS SDK for PHP includes *iterators* that handle the process of sending subsequent requests. Use the
``getIterator()`` method of a client object in order to retrieve an iterator for a particular command.

.. code-block:: php

    $iterator = $client->getIterator('ListObjects', array('Bucket' => 'mybucket'));

    foreach ($iterator as $object) {
        echo $object['Key'] . "\n";
    }

The ``getIterator()`` method accepts either a command object or the name of an operation as the first argument. The
second argument is only used when passing a string and instructs the client on what actual operation to execute.

.. code-block:: php

    $command = $client->getCommand('ListObjects', array('Bucket' => 'mybucket'));
    $iterator = $client->getIterator($command);
