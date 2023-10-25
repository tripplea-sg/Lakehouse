# Setup Resource Principal

On OCI Console, go to menu Dynamic Group and create new Dynamic Group "Lakehouse-dynamicGroup" with the following rule:
```
ALL{resource.type='mysqldbsystem', resource.compartment.id = 'ocid1.compartment.oc1..alphanumericString' }
```
Change compartment_id with your compartment_id </br>
</br>
Go to Policy, create or add the following rules to the existing policy:
```
allow dynamic-group Lakehouse-dynamicGroup to read buckets in compartment Lakehouse-Data
allow dynamic-group Lakehouse-dynamicGroup to read objects in compartment Lakehouse-Data
```
Change Lakehouse-Data with your compartment name </br>
