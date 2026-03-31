### Implement Process Codecs

## Process Info

- process name = `finops2.walletsweep`
- source model module = `domain`
- source model package = `dev.fintech.domain.processes.**`
- codecs module = `infra/serializers`
- codecs package = `dev.fintech.infra.serializers.domain.<process-name>`

## References

- existing codecs at `dev.fintech.infra.serializers.domain.**`

## Notes

- Specs for events codec only, no need for states.
- Json files should be created empty, do not fill them with data, I will fill them later.
- Empty jsons will cause tests failures, so don't run test, run compilation only.
- Json files should be empty, but test cases should be implemented fully.

## Mandatory skills to use

- `circe-codecs`
