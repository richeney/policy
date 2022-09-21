# Policy

Example custom policies for the ALZ hack.

## Create sub files

The standard in policy repos is to have the main JSON files - just the properties - and then have additional files for the parameters and rules, ready for certain CLI commands to use.

```bash
for policy in $(find . -name "policy-*.json" ! -name "*.parameters.json" ! -name "*.rules.json")
do
  jq .parameters $policy > ${policy%%.json}.parameters.json
  jq .policyRule $policy > ${policy%%.json}.rules.json
done
```

## Creating custom policy using the CLI

```bash
policy=./tags/policy-inherit_environment_tag_from_subscription.json
```

```bash
name=$(basename $policy .json | sed 's/^policy-//1')
desc=$(jq -r .description $policy)
display=$(jq -r .displayName $policy)
mode=$(jq -r .mode $policy)
metadata=$(jq -r '.metadata|to_entries|map("\(.key)=\(.value|tostring)")|.[]' $policy)
params=${policy%%.json}.parameters.json
rules=${policy%%.json}.rules.json
```

```bash
# az policy definition create --name $name --description "$desc" --display-name "$display" --metadata $metadata --mode $mode --rules $rules --params $params
az policy definition create --name $name --description "$desc" --display-name "$display" --metadata $metadata --mode $mode --rules $rules
```

Creates at subscription scope. Add `--subscription` to set subscription. Or use `--management-group`.

## Assign

```bash
az policy assignment create --name $name --policy $name --mi-system-assigned --description "$desc" --display-name "$display"
```
