# ironbank_release_action
GitHub Action for automating releases on to [Iron Bank](https://p1.dso.mil/ironbank)

TODO: Inputs, outputs, example, and security warnings
## Input and Output Arguments
### Input
#### `command_string` (Required)

Command string to be executed by SAF CLI. The action will run `saf <command_string>`.

Example:

* `convert asff2hdf -i asff-findings.json -o output-file-name.json`
* More examples can be found at [SAF CLI Usage](https://github.com/mitre/saf#usage).
* NOTE: This action does not support `view heimdall` or input file-type detection (i.e omission of the expected conversion type).

### Output

As determined by input command.

## Secrets

This action does not use any GitHub secrets.

## Example

Below is an example action.

```
on: [push]
jobs:
  saf_hdf_conversion:
    runs-on: ubuntu-latest
    name: SAF CLI Convert ASFF to HDF
    steps:
      - name: Checkout
        uses: actions/checkout@v2
      - name: Convert ASFF
        uses: mitre/saf_action@v1
        with:
          command_string: 'convert asff2hdf -i asff_sample.json -o asff_sample_hdf.json'
      - name: Artifacts
        uses: actions/upload-artifact@v4
        if: success()
        with:
          name: asff
          path: asff_sample_hdf.json
```

For more examples, check out [this workflow](https://github.com/mitre/saf_action/blob/main/.github/workflows/example-usages.yml).

## Contributing, Issues and Support

### Contributing

Please feel free to look through our issues, make a fork, and submit PRs and improvements. We love hearing from our end-users and the community and will be happy to engage with you on suggestions, updates, fixes or new capabilities.

### Issues and Support

Please feel free to contact us by [opening an issue](https://github.com/mitre/saf_action/issues/new/choose) or emailing us at [saf@mitre.org](mailto:saf@mitre.org) should you have any suggestions, questions, or issues.  Please direct security concerns to [saf-security@mitre.org](mailto:saf-security@mitre.org).
