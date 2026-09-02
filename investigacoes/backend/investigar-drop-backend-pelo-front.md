
## Pega vezes que a table veio vazia
```
{app="$environment"}
| json
|= `allTable (id: \\\"\\\")`
```
## Pega vezes que a dataset veio vazia
```
{app="$environment"}
| json
|= `allDataset (id: \\\"\\\")`
```
### Pegar todos os 504 por minito
```
sum(

  count_over_time(

    {app="$environment"}

    | json

    |~ `504`

    [1m]

  )

)
```
## Query bomba

```
query {
    allTable(id: "") {
      edges {
        node {
          _id
          slug
          name
          namePt
          description
          descriptionPt
          isClosed
          version
          isDeprecated
          temporalCoverage
          fullTemporalCoverage
          isDirectory
          auxiliaryFilesUrl
          uncompressedFileSize
          numberRows
          partitions
          publishedByInfo
          dataCleanedByInfo

          status { _id slug }

          dataset {
            _id
            slug
            organizations {
              edges { node { _id slug name namePt } }
            }
          }

          cloudTables {
            edges {
              node {
                gcpTableId
                gcpDatasetId
                gcpProjectId
              }
            }
          }

          rawDataSource {
            edges {
              node {
                _id
                name
                namePt
                dataset { _id }
                polls { edges { node { _id latest } } }
                updates {
                  edges {
                    node {
                      _id
                      latest
                      frequency
                      entity { _id slug }
                    }
                  }
                }
              }
            }
          }

          updates {
            edges {
              node {
                _id
                frequency
                lag
                latest
                entity { _id slug }
              }
            }
          }

          observationLevels {
            edges {
              node {
                _id
                order
                entity { _id name namePt }
                columns {
                  edges { node { _id name namePt } }
                }
              }
            }
          }
        }
      }
  }}
```