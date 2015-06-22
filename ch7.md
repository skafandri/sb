#7. JSON-RPC Server

We are going to be working on the repository that you have created up to this point. If you skipped
ahead, don't worry. You can clone the repository from its Github repository.

````bash
$ git clone git@github.com:skafandri/symfony-tutorial.git --branch ch6
Cloning into 'symfony-tutorial'...
remote: Counting objects: 737, done.
remote: Total 737 (delta 0), reused 0 (delta 0), pack-reused 737
Receiving objects: 100% (737/737), 418.24 KiB | 66.00 KiB/s, done.
Resolving deltas: 100% (378/378), done.
Checking connectivity... done.
````

We are going to implement a JSON-RPC server to expose some of our application's logic as webservices.  
This functionality is clearly not specific to our application.  
For this reason, we will implement it as a stand alone bundle.


##7.1 JsonRpcBundle

We will start to partially implement JSON-RPC 2.0 specifications. For a full specification reference please check http://www.jsonrpc.org/specification

Let's start by creating a new JsonRpcBundle


- Create **src/JsonRpcBundle/JsonRpcBundle.php***

````php
<?php

namespace JsonRpcBundle;

use Symfony\Component\HttpKernel\Bundle\Bundle;

class JsonRpcBundle extends Bundle
{

}
````

To enable the new bundle, edit **app/AppKernel.php** and add `new JsonRpcBundle\JsonRpcBundle()` to the $bundles array.

- Create **src/JsonRpcBundle/Server.php**

````php
<?php

namespace JsonRpcBundle;

use Symfony\Component\DependencyInjection\ContainerAware;
use Symfony\Component\OptionsResolver\OptionsResolver;
use Symfony\Component\Serializer\Encoder\JsonEncoder;
use Symfony\Component\Serializer\Exception\UnexpectedValueException;

class Server extends ContainerAware
{

    const ID = 'json_rpc.server';

    public function handle($request, $serviceId)
    {
        $encoder = new JsonEncoder();
        try {
            $request = $encoder->decode($request, JsonEncoder::FORMAT);
        } catch (UnexpectedValueException $exception) {
            return new ErrorResponse(ErrorResponse::ERROR_CODE_PARSE_ERROR, 'Invalid JSON');
        }

        $request = $this->resolveOptions($request);

        if (!$this->isAllowed($serviceId, $request['method'])) {
            return new ErrorResponse(
                    ErrorResponse::ERROR_CODE_METHOD_NOT_FOUND,
                    sprintf('%s does not exist', $request['method'])
            );
        }

        $service = $this->container->get($serviceId);
        $result = call_user_func_array(
                array(
                    $service,
                    $request['method']
                ),
                $request['params']
        );

        return new SuccessResponse($request['id'], $result);
    }

    private function isAllowed($serviceId, $method)
    {
        return true;
    }

    private function resolveOptions($request)
    {
        $resolver = new OptionsResolver();
        $resolver
                ->setRequired('id')
                ->setRequired('method')
                ->setRequired('jsonrpc')
                ->setDefault('params', array())
                ->addAllowedValues('jsonrpc', Response::VERSION);
        return $resolver->resolve($request);
    }

}
````

- Create **src/JsonRpcBundle/Resources/config/services.yml**

````
services:
    json_rpc.server:
        class: JsonRpcBundle\Server
        calls:
            - [setContainer, ["@service_container"]]
````

- Create **src/JsonRpcBundle/DependencyInjection/JsonRpcExtension.php**

````php
<?php

namespace JsonRpcBundle\DependencyInjection;

use Symfony\Component\Config\FileLocator;
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\DependencyInjection\Extension\Extension;
use Symfony\Component\DependencyInjection\Loader\YamlFileLoader;

class JsonRpcExtension extends Extension
{
    public function load(array $config, ContainerBuilder $container)
    {
        $loader = new YamlFileLoader($container, new FileLocator(__DIR__.'/../Resources/config'));
        $loader->load('services.yml');
    }
}
````

- Create **src/JsonRpcBundle/Response.php**

````php
<?php

namespace JsonRpcBundle;

abstract class Response
{

    const VERSION = '2.0';

    private $id;
    private $jsonrpc = self::VERSION;
    private $resultKey;
    private $result;

    public function __construct($id, $resultKey, $result)
    {
        $this->id = $id;
        $this->resultKey = $resultKey;
        $this->result = $result;
    }

    public function getId()
    {
        return $this->id;
    }

    public function getJsonrpc()
    {
        return $this->jsonrpc;
    }

    public function getResultKey()
    {
        return $this->resultKey;
    }

    public function getResult()
    {
        return $this->result;
    }

    public function setId($id)
    {
        $this->id = $id;
        return $this;
    }

    public function setJsonrpc($jsonrpc)
    {
        $this->jsonrpc = $jsonrpc;
        return $this;
    }

    public function setResultKey($resultKey)
    {
        $this->resultKey = $resultKey;
        return $this;
    }

    public function setResult($result)
    {
        $this->result = $result;
        return $this;
    }

    public function toArray()
    {
        return array(
            'id' => $this->getId(),
            'jsonrpc' => $this->getJsonrpc(),
            $this->getResultKey() => $this->getResult()
        );
    }

}
````

- Create **src/JsonRpcBundle/SuccessResponse.php**

````php
<?php

namespace JsonRpcBundle;

class SuccessResponse extends Response
{

    public function __construct($id, $result)
    {
        parent::__construct($id, 'result', $result);
    }

}
````

- Create **src/JsonRpcBundle/ErrorResponse.php**

````php
<?php

namespace JsonRpcBundle;

class ErrorResponse extends Response
{
    const ERROR_CODE_PARSE_ERROR = -32700;
    const ERROR_CODE_METHOD_NOT_FOUND = -32601;
    const ERROR_CODE_SERVER_ERROR = -32000;

    public function __construct($code, $message, $id = null)
    {
        parent::__construct($id, 'error', array('code' => $code, 'message' => $message));
    }

}
````

- Create **src/JsonRpcBundle/Exception/JsonRpcException.php**

````php
<?php

namespace JsonRpcBundle\Exception;

class JsonRpcException extends \Exception
{

}
````

- Create **src/JsonRpcBundle/Exception/InvalidMethodException.php**

````php
<?php

namespace JsonRpcBundle\Exception;

class InvalidMethodException extends JsonRpcException
{

}
````

- Create **src/JsonRpcBundle/Controller/ServerController.php**

````php
<?php

namespace JsonRpcBundle\Controller;

use JsonRpcBundle\Server;
use Symfony\Bundle\FrameworkBundle\Controller\Controller;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpFoundation\Request;

class ServerController extends Controller
{

    public function handleAction(Request $request, $service)
    {
        $server = $this->get(Server::ID);
        $result = $server->handle($request->getContent(), $service);
        return new JsonResponse($result->toArray());
    }

}
````

- Create **src/JsonRpcBundle/Resources/config/routing.yml**

````
json_rpc:
    path:     /json-rpc/{service}
    defaults: { _controller: "JsonRpcBundle:Server:handle" }
    methods: [POST]
````

- Edit **app/config/routing.yml** and add the fowlling route

````
json_rpc:
    resource: "@JsonRpcBundle/Resources/config/routing.yml"
````

- Edit **src/AppBundle/Service/WarehouseService.php** update getAll method

````php
public function getAll()
{
    return $this->entityManager
            ->createQueryBuilder()
            ->select('warehouse')
            ->from(Warehouse::REPOSITORY, 'warehouse')
            ->getQuery()
            ->getArrayResult();
}
````

Done, you can check if this webservice is working by posting

````
{
  "jsonrpc": "2.0",
  "method": "getAll",  
  "id": 1
}
````
to http://127.0.0.1:8000/json-rpc/app.warehouse

You should get a response similar to

````
{
    "id": 1,
    "jsonrpc": "2.0",
    "result": [
        {
            "id": 1,
            "name": "warehouse 1",
            "address": null
        },
        {
            "id": 2,
            "name": "warehouse 2",
            "address": null
        }
    ]
}
````

## 7.2 Service tagging

The bundle we just created will expose any method from any service available in our application.

If you try for example to post

````
{
  "jsonrpc": "2.0",
  "method": "trans",
  "params": ["order.status.20"],
  "id": 1
}
```
to http://127.0.0.1:8000/json-rpc/translator you will get

````
{
    "id": 1,
    "jsonrpc": "2.0",
    "result": "delivered"
}
````

The `isAllowed` method from our server service just returns true. We will update this logic to effectively check if the method is allowed.

- Edit **src/JsonRpcBundle/Server.php**

Add  private $allowedMethods = array();

Update isAllowed method

````php
private function isAllowed($serviceId, $method)
{
    return in_array(sprintf('%s->%s', $serviceId, $method), $this->allowedMethods);
}
````

Add addAllowedMethod

````php
public function addAllowedMethod($serviceId, $method)
{
    $this->allowedMethods[] = sprintf('%s->%s', $serviceId, $method);
}
````

Now we need a way to gather the list of serviceId/method.

We will define a custom tag [json_rpc.service] and add it to each service method we want to enable.

- Edit **src/AppBundle/Resources/config/services.yml** and update app.warehouse definition

````
app.warehouse:
        class: AppBundle\Service\WarehouseService
        parent: app.doctrine_aware
        tags:
            - {name: json_rpc.service, method: getAll}
````

To parse all the services tagged with our custom tag, we will need to write a custom compiler pass.

- Create **src/JsonRpcBundle/DependencyInjection/Compiler/ServicePass.php**

````php
<?php

namespace JsonRpcBundle\DependencyInjection\Compiler;

use Symfony\Component\DependencyInjection\Compiler\CompilerPassInterface;
use Symfony\Component\DependencyInjection\ContainerBuilder;

class ServicePass implements CompilerPassInterface
{

    public function process(ContainerBuilder $container)
    {
        $serverDefinition = $container->findDefinition(\JsonRpcBundle\Server::ID);
        $resolvers = $container->findTaggedServiceIds('json_rpc.service');

        foreach ($resolvers as $id => $tagAttributes) {
            foreach ($tagAttributes as $attributes) {
                $method = $attributes['method'];
                $serverDefinition->addMethodCall('addAllowedMethod', array($id, $method));
            }
        }
    }

}
````

Now we need to the previous compiler pass when building our bundle.

- Edit **src/JsonRpcBundle/JsonRpcBundle.php**

````php
<?php

namespace JsonRpcBundle;

use JsonRpcBundle\DependencyInjection\Compiler\ServicePass;
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\HttpKernel\Bundle\Bundle;

class JsonRpcBundle extends Bundle
{
    public function build(ContainerBuilder $container)
    {
        parent::build($container);
        $container->addCompilerPass(new ServicePass());
    }

}
````

Done, clear your cache, now only app.warehouse->getAll is accessible as a json-rpc service.

When our application will grow and we will have many services exposed as json-rpc services, we can inspect the methods list by running

````bash
$ app/console debug:container --tag=json_rpc.service
[container] Public services with tag json_rpc.service
Service ID    method Class name
app.warehouse getAll AppBundle\Service\WarehouseService
````

##7.3 Json-rpc services list

The new service works as expected, but when we will have many services enabled as json-rpc services, it will be hard to localize all of them.
