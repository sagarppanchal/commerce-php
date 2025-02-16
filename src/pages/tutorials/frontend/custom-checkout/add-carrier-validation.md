---
title: Add Custom Shipping Carrier Validations | Commerce PHP Extensions
description: Follow this tutorial to create programmatic validations for custom shipping carriers in the Adobe Commerce and Magento Open Source checkout experience.
---

# Add custom shipping carrier validations

This topic describes how to add shipping address validations for a custom shipping carrier to the checkout. This is an essential part of the bigger task of adding a custom shipping method to your store.

To add new shipping carrier validations to the checkout, do the following:

1. [Create validation rules](#step-1-create-validation-rules).
1. [Create validator](#step-2-create-validator).
1. [Register validator and rules in the validators pool](#step-3-register-validator-and-rules-in-the-validators-pool).
1. [Add the validators and rules to the checkout layout](#step-4-add-the-validation-to-the-checkout-layout).

<InlineAlert variant="info" slots="text"/>

During checkout, when a customer fills the shipping address form, shipping carrier validations trigger the shipping rates request. That is why adding shipping carrier validations for your custom shipping method is mandatory.

## Step 1: Create validation rules

Shipping carrier validation rules declare which fields of the shipping address are required for the corresponding shipping method to be available. The validation itself is performed by the [validator](#step-2-create-validator).

During checkout, if the shipping address fields declared in the rules are filled, the further validation of fields' values is carried on the server side. For example, whether a carrier is available for the specified country.

For the sake of compatibility, upgradability, and easy maintenance, do not edit the default application code, add your customization in a separate module. For your checkout customization to be applied correctly, your custom module should [depend](../../../development/build/composer-integration.md) on the `Magento_Checkout` module.

Do not use `Ui` for your custom module name, because `%Vendor%_Ui` notation, required when specifying paths, might cause issues.

In your `<your_module_dir>/view/frontend/web/js/model` directory, create a `.js` file implementing the validation rules.

The script must implement the `getRules()` method.

For example, the FedEx shipping method requires only two fields of the shipping address to be filled: **Country**, **Zip Code** and **City**. This is how the validation rules for FedEx look:

> _<Magento_Fedex_dir>/view/frontend/web/js/model/shipping-rates-validation-rules.js_

```js
define(
    [],
    function () {
        'use strict';
        return {
            getRules: function() {
                return {
                    'postcode': {
                        'required': true
                    },
                    'country_id': {
                        'required': true
                    },
                    'city': {
                        'required': true
                    }
                };
            }
        };
    }
)
```

Triggering the shipping rates request correlates directly with the fields you specify in the validation rules: the request is triggered once all these fields are populated and pass the validation.

## Step 2: Create validator

Create the validator `.js` script that checks if the fields defined by the validation rules are filled. The script must be located in the `<your_module_dir>/view/frontend/web/js/model` directory.

A sample validator script follows:

```js
define(
    [
        'jquery',
        'mageUtils',

        './shipping-rates-validation-rules',
        'mage/translate'
    ],
    function ($, utils, validationRules, $t) {
        'use strict';
        return {
            validationErrors: [],
            validate: function(address) {
                var self = this;
                this.validationErrors = [];
                $.each(validationRules.getRules(), function(field, rule) {
                    if (rule.required && utils.isEmpty(address[field])) {
                        var message = $t('Field ') + field + $t(' is required.');
                        self.validationErrors.push(message);
                    }
                });
                return !Boolean(this.validationErrors.length);
            }
        };
    }
);
```

You can use this sample for your validator, but you need to specify your validation rules script instead of `./shipping-rates-validation-rules` in the list of used modules.

## Step 3: Register validator and rules in the validators pool

Your custom validator must be added to the pool of validators. To do this, in the `<your_module_dir>/view/frontend/web/js/view` directory create a new `<your-validation>.js` file with the following content (having replaced the `<your_validator>` and `<your_validation_rules>` with your values):

```js
define(
    [
        'uiComponent',
        'Magento_Checkout/js/model/shipping-rates-validator',
        'Magento_Checkout/js/model/shipping-rates-validation-rules',
        '../model/<your_validator>',
        '../model/<your_validation_rules>'
    ],
    function (
        Component,
        defaultShippingRatesValidator,
        defaultShippingRatesValidationRules,
        shippingRatesValidator,
        shippingRatesValidationRules
    ) {
        'use strict';
        defaultShippingRatesValidator.registerValidator('carrierName', shippingRatesValidator);
        defaultShippingRatesValidationRules.registerRules('carrierName', shippingRatesValidationRules);
        return Component;
    }
);
```

## Step 4: Add the validation to the checkout layout

The last step is specifying the script you created on the previous step in the checkout page layout.

In your custom module directory, create a new `<your_module_dir>/view/frontend/layout/checkout_index_index.xml` file.

In this file, add the following:

You must add `<your-validation-name>` like `%carrier%-rates-validation` - where carrier has to match the actual carrier code.

```xml
<page xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="urn:magento:framework:View/Layout/etc/page_configuration.xsd">
    <body>
        <referenceBlock name="checkout.root">
            <arguments>
                <argument name="jsLayout" xsi:type="array">
                    <item name="components" xsi:type="array">
                        <item name="checkout" xsi:type="array">
                            <item name="children" xsi:type="array">
                                <item name="steps" xsi:type="array">
                                    <item name="children" xsi:type="array">
                                        <item name="shipping-step" xsi:type="array">
                                            <item name="children" xsi:type="array">
                                                <item name="step-config" xsi:type="array">
                                                    <item name="children" xsi:type="array">
                                                        <item name="shipping-rates-validation" xsi:type="array">
                                                            <item name="children" xsi:type="array">
                                                                <item name="<your-validation-name>" xsi:type="array">
                                                                    <item name="component" xsi:type="string">%your_module%/js/view/%your-validation%</item>
                                                                </item>
                                                            </item>
                                                        </item>
                                                    </item>
                                                </item>
                                            </item>
                                        </item>
                                    </item>
                                </item>
                            </item>
                        </item>
                    </item>
                </argument>
            </arguments>
        </referenceBlock>
    </body>
</page>
```

## Step 5: Deploy static content and clear the cache

1. Deploy static content:

```bash
bin/magento setup:static-content:deploy
```

1. Clean the cache:

```bash
bin/magento cache:clean
```
