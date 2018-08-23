# swoole-bridge-bundle
Symfony Swoole Bridge Bundle

## Requirements

* PHP >= 7.2
* symfony/framework-bundle: "^3.0|^4.0"
* insidestyles/swoole-bridge: "^0.1"
* symfony/console: "^3.0|^4.0",
* zendframework/zend-diactoros: "^1.7"


## Installation

This package is installable and autoloadable via Composer 

```sh
composer require insidestyles/swoole-bridge-bundle
```
Update config.yml
```yaml
swoole_bridge:
    server:
        port: "%web_server_port%"      #The port the socket server will listen on
        host: "%web_server_host%"
```
Update AppKernel
```php
    new Insidestyles\SwooleBridgeBundle\SwooleBridgeBundle(),
```

## Usage

```sh
    php bin/console swoole:bridge:server
```

## Create custom handler
One of common problems while running long lived program in php is "MySQL server has gone away" (Long lived php 
daemon running using reactphp, websocket, swoole, ..etc..). 
In this case we can create a custom handler to handle this error. For example:

```php
class CustomHandler implements SwooleBridgeInterface
{    
    /**
     * @var SwooleAdapterInterface
     */
    private $adapter;

    /**
     * @var RegistryInterface
     */
    private $doctrineRegistry;

    /**
     * @var LoggerInterface
     */
    private $logger;

    /**
     * Handler constructor.
     * @param SwooleAdapterInterface $adapter
     * @param null|LoggerInterface $logger
     */
    public function __construct(
        SwooleAdapterInterface $adapter,
        RegistryInterface $doctrineRegistry,
        ?LoggerInterface $logger = null
    ) {
        $this->adapter = $adapter;
        $this->doctrineRegistry = $doctrineRegistry;
        $this->logger = $logger ?? new NullLogger();
    }

    /**
     * @inheritdoc
     */
    public function handle(
        SwooleRequest $swooleRequest,
        SwooleResponse $swooleResponse
    ): void {
        try {
            $this->adapter->handle($swooleRequest, $swooleResponse);
        } catch (PDOException $e) {
            $this->logger->error($e->getMessage());
            /** @var EntityManagerInterface $entityManager */
            $entityManager = $this->doctrineRegistry->getManager();
            if(!$entityManager->isOpen()) {
                $this->logger->warning('Resetting entity manager');
                $this->doctrineRegistry->resetManager();
            }
        } catch (\Throwable $e) {
            $this->logger->error($e->getMessage());
        }
    }
}
    
```

Override default handler using symfony service decorator:
 
```yml 
#config/services.yaml
services:
    swoole_bridge.handler:
        class: App\CustomHandler
        decorates: swoole_bridge.handler
        arguments: ['@swoole_bridge.handler.inner']
