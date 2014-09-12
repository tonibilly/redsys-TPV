Redsys
=====

Este script te permitirá generar los formularios para la integración de la pasarela de pago de Redsys (antes Sermepa / Servired).

## Ejemplo de pago instantáneo

Este proceso se realiza para pagos en el momento, sin necesidad de confirmación futura (TransactionType = 0)

```php
include (__DIR__.'/libs/ANS/Redsys/Redsys.php');

# Cargamos la clase con los parámetros base

$Redsys = new \ANS\Redsys\Redsys(array(
    'Environment' => 'test', // Puedes indicar test o real
    'MerchantCode' => '1234567890',
    'Key' => 'asdfghjkd0123456789',
    'Terminal' => '1',
    'Currency' => '978',
    'MerchantName' => 'COMERCIO',
    'Titular' => 'Mi Comercio',
    'Currency' => '978',
    'Terminal' => '1',
    'ConsumerLanguage' => '001'
));

# Indicamos los campos para el pedido

$Redsys->setFormHiddens(array(
    'TransactionType' => '0',
    'MerchantData' => 'Televisor de 50 pulgadas',
    'Order' => '012121323',
    'Amount' => '568,25',
    'UrlOK' => 'http://dominio.com/direccion-todo-correcto/',
    'UrlKO' => 'http://dominio.com/direccion-error',
    'MerchantURL' => 'http://dominio.com/direccion-control-pago'
));

# Imprimimos el pedido el formulario y redirigimos a la TPV

echo '<form action="'.$TPV->getPath('/realizarPago').'" method="post">'.$TPV->getFormHiddens().'</form>';

die('<script>document.forms[0].submit();</script>');
```

Para realizar el control de los pagos, la TPV se comunicará con nosotros a través de la url indicada en **MerchantURL**.

Este script no será visible ni debe responder nada, simplemente verifica el pago.

El banco siempre se comunicará con nosotros a través de esta url, sea correcto o incorrecto.

Podemos realizar un script (Lo que en el ejemplo sería http://dominio.com/direccion-control-pago) que valide los pagos de la siguiente manera:

```php
include (__DIR__.'/libs/ANS/Redsys/Redsys.php');

# Cargamos la clase con los parámetros base

$Redsys = new \ANS\Redsys\Redsys(array(
    'Environment' => 'test', // Puedes indicar test o real
    'MerchantCode' => '1234567890',
    'Key' => 'asdfghjkd0123456789',
    'Terminal' => '1',
    'Currency' => '978',
    'MerchantName' => 'COMERCIO',
    'Titular' => 'Mi Comercio',
    'Currency' => '978',
    'Terminal' => '1',
    'ConsumerLanguage' => '001'
));

# Realizamos la comprobación de la transacción

try {
    $Redsys->checkTransaction($_POST);
} catch (\Exception $e) {
    file_put_contents(__DIR__.'/logs/errores-tpv.log', $e->getMessage(), FILE_APPEND);
    die();
}

# Actualización del registro en caso de pago (ejemplo usando mi framework)

$Db->update(array(
    'table' => 'tpv',
    'limit' => 1,
    'data' => array(
        'operacion' => $_POST['Ds_TransactionType'],
        'fecha_pago' => date('Y-m-d H:i:s')
    ),
    'conditions' => array(
        'id' => $_POST['Ds_Order']
    )
));

die();
```

## Ejemplo de pago en diferido

Este proceso se realiza para pagos mediante autorización inicial y posterior confirmación del pago sin que el cliente se encuentre presente (TransactionType = 1)

El proceso es exactamente igual que el anterior, sólamente se debe cambiar el valor de inicalización de `TransactionType` de `0` a `1`.

Una vez completado todo el proceso anterior, debemos crear dos scripts en nuestro proyecto, uno para iniciar la confirmación del pago y otro para verificar el proceso.

```php
include (__DIR__.'/libs/ANS/Redsys/Redsys.php');

# Cargamos la clase con los parámetros base

$Redsys = new \ANS\Redsys\Redsys(array(
    'Environment' => 'test', // Puedes indicar test o real
    'MerchantCode' => '1234567890',
    'Key' => 'asdfghjkd0123456789',
    'Terminal' => '1',
    'Currency' => '978',
    'MerchantName' => 'COMERCIO',
    'Titular' => 'Mi Comercio',
    'Currency' => '978',
    'Terminal' => '1',
    'ConsumerLanguage' => '001'
));

# Indicamos los campos para la confirmación del pago

$Redsys->sendXml([
    'TransactionType' => '2', // Código para la Confirmación del cargo
    'MerchantURL' => 'http://dominio.com/direccion-control-pago-xml', // A esta URL enviará el banco la confirmación del cobro
    'Amount' => '568,25', // La cantidad final a cobrar
    'Order' => '012121323', // El número de pedido, que debe existir en el sistema bancario a través de una autorización previa
    'MerchantData' => 'Televisor de 50 pulgadas',
]));
````

Esta ejecución nos devolverá un XML con una respuesta sobre este envío, pero la respuesta sobre el resultado de la operación serán enviada desde el banco a la URL indicada en MerchantURL.

Para verificar que el envío se ha realizado correctamente, el banco devuelve un XML con un valor para la etiqueta de  de `CODIGO` que devemos verificar para saber si el envío ha sido correcto.

Ahora vamos a por el script de `http://dominio.com/direccion-control-pago-xml` en que recogemos el resultado del pago:

```php
include (__DIR__.'/libs/ANS/Redsys/Redsys.php');

# Cargamos la clase con los parámetros base

$Redsys = new \ANS\Redsys\Redsys(array(
    'Environment' => 'test', // Puedes indicar test o real
    'MerchantCode' => '1234567890',
    'Key' => 'asdfghjkd0123456789',
    'Terminal' => '1',
    'Currency' => '978',
    'MerchantName' => 'COMERCIO',
    'Titular' => 'Mi Comercio',
    'Currency' => '978',
    'Terminal' => '1'
));

# Obtenemos los datos remitidos por el banco en formato `array`

$datos = $Redsys->xmlString2array($_POST['datos']);

# Realizamos la comprobación de la transacción

try {
    $Redsys->checkTransaction($datos);
} catch (\Exception $e) {
    file_put_contents(__DIR__.'/logs/errores-tpv.log', $e->getMessage(), FILE_APPEND);
    die();
}

# Actualización del registro en caso de pago (ejemplo usando mi framework)

$Db->update(array(
    'table' => 'tpv',
    'limit' => 1,
    'data' => array(
        'pagado' => 1,
        'operacion' => $datos['Ds_TransactionType'],
        'fecha_pago' => date('Y-m-d H:i:s')
    ),
    'conditions' => array(
        'id' => $datos['Ds_Order']
    )
));

die();
```

--------

Una manera más elegante sería guardando la configuración en un fichero llamado por ejemplo `config.php` e incluirlo directamente en la carga de la clase:

```php
return array(
    'Environment' => 'test', // Puedes indicar test o real
    'MerchantCode' => '1234567890',
    'Key' => 'asdfghjkd0123456789',
    'Terminal' => '1',
    'TransactionType' => '0',
    'Currency' => '978',
    'MerchantName' => 'COMERCIO',
    'Titular' => 'Mi Comercio',
    'Currency' => '978',
    'Terminal' => '1',
    'ConsumerLanguage' => '001'
);
```

y así incluimos directamente el fichero y evitamos ensuciar el script con líneas de configuración

```php
include (__DIR__.'/libs/ANS/Redsys/Redsys.php');

$Redsys = new \ANS\Redsys\Redsys(require(__DIR__.'/config.php'));
```

Para gustos, colores :)

Si deseas más información sobre parámetros u opciones, Google puede echarte una mano https://www.google.es/search?q=manual+instalaci%C3%B3n+redsys+php+filetype%3Apdf
