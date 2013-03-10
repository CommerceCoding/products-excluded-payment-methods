# Products Excluded Payment Methods

## Wichtig

Diese Erweiterung wurde auf Basis der modified eCommerce Shopsoftware (Version 1.06 rev 4356) entwickelt. Alle Dateien dieser Erweiterung ergänzen vorhandene Dateien des Shopsystems.  Die Dateien sollten nicht einfach über bestehende kopiert werden, sondern sollen als Anhaltspunkt für die richtigen Stellen der Einfügen dienen. Ein Backup des Shops vorher ist empfehlenswert. Es wird keine Haftung übernommen.
 
## Datenbankanpassungen

Damit die ausgeschlossenen Zahlungsarten für jedes Produkt gespeichert werden können, ist es notwendig für die Tabelle *products* eine weitere Spalte einzufügen. Dies kann entweder durch den folgenden SQL-Befehl erfolgen:

```sql
ALTER TABLE `products` ADD `products_excluded_payment_methods` VARCHAR(255) NOT NULL DEFAULT 'a:0:{}';
```

Oder es kann mithilfe eines GUI wie z.B. phpMyAdmin geschehen. Wichtig ist dabei der Standardwert *'a:0:{}'*, der für bestehende Produkte gesetzt wird. Es handelt sich dabei um ein serialisiertes leeres Array. 

## Admin-Bereich

### admin/includes/modules/new_product.php

Zur Darstellung eines Auswahlbereichs mit den aktiven Zahlungsarten bei dem Erstellen bzw. Ändern von Produkten, muss in dieser Datei zuerst eine Auflistung der verfügbaren Bezahlmethoden vorbereitet werden. Dies kann überall im ersten PHP-Abschnitt geschehen.

```php
// Products Excluded Payment Methods - Commerce Coding - BEGIN
$paymentMethods = array();
$installedPayments = explode(';', MODULE_PAYMENT_INSTALLED);
foreach ($installedPayments as $installedPayment) {
  if(file_exists(DIR_FS_LANGUAGES . $_SESSION['language'] . '/modules/payment/' . $installedPayment)) {
    require_once DIR_FS_LANGUAGES . $_SESSION['language'] . '/modules/payment/' . $installedPayment;
  }
  if(file_exists(DIR_FS_CATALOG_MODULES . 'payment/' . $installedPayment)) {
    require_once DIR_FS_CATALOG_MODULES . 'payment/' . $installedPayment;
  }
  $class = str_replace(".php", "", $installedPayment);
  if(xtc_class_exists($class)) {
    $payment = new $class();
    if($payment->check()) {
      $paymentMethods[] = array('code' => $payment->code, 'title' => $payment->title);
    }
  }
}
// Products Excluded Payment Methods - Commerce Coding - END
```

Über eine zweite Einfügung in der Datei wird der Formularteil mit der Anzeige der Bezahlmethoden erzeugt. Dies kann z.B. innerhalb der zweiten inneren Tabelle über dem Info-Template-Auswahlfeld erfolgen.

```php
<!-- Products Excluded Payment Methods - Commerce Coding - BEGIN -->
<tr>
  <td style="vertical-align: top;"><span class="main"><?php echo TEXT_EXCLUDE_PAYMENTS; ?>:</span></td>
  <td><span class="main">
    <?php
      if(!isset($pInfo->products_excluded_payment_methods)) {
        $pInfo->products_excluded_payment_methods = 'a:0:{}';
      }
      foreach ($paymentMethods as $payment) {
        echo xtc_draw_selection_field('products_excluded_payment_methods[]', 'checkbox', $payment['code'], in_array($payment['code'], unserialize($pInfo->products_excluded_payment_methods))) . $payment['title'] . '<br>';
      }
    ?>
  </span></td>
</tr><!-- Products Excluded Payment Methods - Commerce Coding - END -->
```

### admin/includes/classes/categories.php

In dieser Datei wird die Eingabe verarbeitet. Die Einfügungen erfolgen in der Funktion *insert_product*. Zuerst werden Eingaben ohne ausgeschlossene Bezahlmethoden abgefangen. Dies kann überall oberhalb des ersten Vorkommens von *$sql_data_array* geschehen.

```php
// Products Excluded Payment Methods - Commerce Coding - BEGIN
if(!is_array($products_data['products_excluded_payment_methods'])) {
  $products_data['products_excluded_payment_methods'] = array();
}
// Products Excluded Payment Methods - Commerce Coding - END
```

Das besagte *$sql_data_array* muss außerdem noch um eine Zeile ergänzt werden, damit die Eingabe auch in der Datenbank gespeichert wird.

```php
'products_excluded_payment_methods' => xtc_db_prepare_input(serialize($products_data['products_excluded_payment_methods']))
```

## Artikelansicht

### includes/modules/product_info.php

Um die ausgeschlossenen Bezahlverfahren den Kunden anzuzeigen, werden ähnlich der Auflistung im Administrationsbereich alle aktivierten Zahlungsarten aufgerufen. Diese werden dahingehend überprüft, ob sie für das gewählte Produkt ausgeschlossen sind. Ist dies der Fall, werden sie dem Template übergeben.

```php
// Products Excluded Payment Methods - Commerce Coding - BEGIN
$excludedPaymentMethods = array();
$installedPayments = explode(';', MODULE_PAYMENT_INSTALLED);
foreach ($installedPayments as $installedPayment) {
  if(file_exists(DIR_FS_CATALOG . 'lang/' . $_SESSION['language'] . '/modules/payment/' . $installedPayment)) {
    require_once DIR_FS_CATALOG . 'lang/' . $_SESSION['language'] . '/modules/payment/' . $installedPayment;
  }
  if(file_exists(DIR_FS_CATALOG . 'includes/modules/payment/' . $installedPayment)) {
    require_once DIR_FS_CATALOG . 'includes/modules/payment/' . $installedPayment;
  }
  $class = str_replace(".php", "", $installedPayment);
  if(class_exists($class)) {
    $payment = new $class();
    if($payment->check() && in_array($payment->code, unserialize($product->data['products_excluded_payment_methods']))) {
      $excludedPaymentMethods[] = $payment->title;
    }
  }
}
$info_smarty->assign('PRODUCTS_EXCLUDED_PAYMENT_METHODS', $excludedPaymentMethods);
// Products Excluded Payment Methods - Commerce Coding - END
```

### templates/xtc5/module/product_info/product_info_v1.html

Im gewünschten Template selbst wird die Anzeige der ausgeschlossenen Zahlungsarten davon abhängig gemacht, ob Daten übergeben worden. Sofern dies der Fall ist, werden sie aufgelistet.

An dieser Stelle besteht natürlich gestalterische Freiheit. Weiterhin können auch die anderen product_info-Templates einfach angepasst werden.

```html
<!-- Products Excluded Payment Methods - Commerce Coding - BEGIN -->
{if $PRODUCTS_EXCLUDED_PAYMENT_METHODS}
<p class="taxandshippinginfo" style="white-space:nowrap">
{#text_excluded_payments#}<br/>
{foreach item=payment from=$PRODUCTS_EXCLUDED_PAYMENT_METHODS}
- {$payment}<br/>
{/foreach}
</p>
{/if}
<!-- Products Excluded Payment Methods - Commerce Coding - END -->
```

## Zahlungsauswahl im Checkout

### includes/classes/shopping_cart.php

Die Funktion *get_products* muss an zwei Stellen angepasst werden. Die Select-Abfrage *$products_query* wird um eine Variable ergänzt.

```php
p.products_excluded_payment_methods
```

Das abgefragte Attribut muss dann noch in das Array *$products_array* übergeben werden.

```php
'excluded_payment_methods' => unserialize($products['products_excluded_payment_methods'])
```

### checkout_payment.php

Bevor dem Kunden die Zahlungsarten zur Auswahl angeboten werden, werden alle heraus gefiltert, welche durch die Produktauswahl ausgeschlossen wurden. Dies erfolgt nach dem Zuweisen der Zahlungsmethoden in die Variable *$selection*.

```php
// Products Excluded Payment Methods - Commerce Coding - BEGIN
$excludedPayment = array();
$products = $_SESSION['cart']->get_products();
foreach($products as $product) {
  foreach($product['excluded_payment_methods'] as $excluded){
    if(!in_array($excluded,$excludedPayment)) {
      $excludedPayment[] = $excluded;
    }
  }
}
foreach($selection as $key => $value) {
  if(in_array($value['id'], $excludedPayment)) {
    unset($selection[$key]);
  }
}
$selection = array_values($selection);
// Products Excluded Payment Methods - Commerce Coding - END
```

## Sprachdateien

### lang/english/lang_english.conf

An den Block *[product_info]* folgendes anfügen:

```
# Products Excluded Payment Methods - Commerce Coding - BEGIN
text_excluded_payments = 'Excluded Payment Methods:'
# Products Excluded Payment Methods - Commerce Coding - END
```

### lang/english/admin/categories.php

An die Datei folgendes anfügen:

```php
// Products Excluded Payment Methods - Commerce Coding - BEGIN
define('TEXT_EXCLUDE_PAYMENTS','Exclude Payment Methods');
// Products Excluded Payment Methods - Commerce Coding - END
```

### lang/german/lang_german.conf

An den Block *[product_info]* folgendes anfügen:

```
# Products Excluded Payment Methods - Commerce Coding - BEGIN
text_excluded_payments = 'Ausgeschlossene Zahlungsarten:'
# Products Excluded Payment Methods - Commerce Coding - END
```

### lang/german/admin/categories.php

An die Datei folgendes anfügen:

```php
// Products Excluded Payment Methods - Commerce Coding - BEGIN
define('TEXT_EXCLUDE_PAYMENTS','Zahlungsarten ausschlie&szlig;en');
// Products Excluded Payment Methods - Commerce Coding - END
```

## Credits
Copyright: © 2013, Commerce Coding, http://www.commerce-coding.de  
Lizenz: http://opensource.org/licenses/GPL-2.0  GNU General Public License, version 2 (GPL-2.0)