# Monitor the logs

Percona XtraDB Cluster, built on the Galera Cluster technology, offers robust database server logging features. By default, Percona XtraDB Cluster logs errors to a `<hostname>.err` file located in the data directory. This logging behavior can be customized in the `my.cnf` configuration file by using the `log_error` option or by specifying the `--log-error` parameter. This flexibility ensures that you can manage error logging effectively to suit your operational needs.
