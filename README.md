# Scoop Ao

[![CI](https://github.com/YangAoLib/scoop-ao/actions/workflows/ci.yml/badge.svg)](https://github.com/YangAoLib/scoop-ao/actions/workflows/ci.yml)
[![Excavator](https://github.com/YangAoLib/scoop-ao/actions/workflows/excavator.yml/badge.svg)](https://github.com/YangAoLib/scoop-ao/actions/workflows/excavator.yml)

A personal [Scoop](https://scoop.sh) bucket maintained by YangAoLib.

## Usage

```powershell
scoop bucket add ao https://github.com/YangAoLib/scoop-ao
scoop install ao/kook
scoop install ao/benchlocal
```

## Manifests

| Name | Description |
| --- | --- |
| [kook](https://www.kookapp.cn) | Voice communication tool |
| [benchlocal](https://github.com/stevibe/BenchLocal) | 本地优先的大语言模型基准测试与模型对比桌面应用 |

## Maintenance

GitHub Actions validates manifests on every change. Excavator checks for application updates every four hours and updates manifests automatically.

The KOOK manifest resolves the vendor's short-lived signed download URL through the open-source [dorado-api](https://github.com/chawyehsu/dorado-api) redirect service. The installer package is downloaded directly from KOOK and verified against the SHA-256 stored in the manifest.

## License

This bucket is released under the [Unlicense](LICENSE). Applications installed by this bucket retain their respective licenses.
