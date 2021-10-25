# Deployment

The deployment of modules is straight forward - simply drop the module files in the apropriate directory.

For modules that are part of the default installation, they're located in the following directory (assuming the default path for your cerebrate installation:

```
/var/www/cerebrate/src/Lib/default/local\_tool\_connectors
```

If you wish to deploy your own custom modules, they'd be installed by dropping them in:

```
/var/www/cerebrate/src/Lib/custom/local\_tool\_connectors
```

## File and class naming conventions

The naming convention for the Cerebrate integration modules dictates that each new modules has to be named MyModuleConnector.php . Be mindful of the capitalisation, which is expected to be PascalCase.

The classname has to follow the naming convention, so for the above example, our class name would be MyModuleConnector.

## File permissions

Make sure that your file is readable by the apache user (www-data or apache normally)

```
chown www-data:www-data /var/www/cerebrate/sc/Lib/custom/local\_tool\_connectors/MyModuleConnector.php
```


