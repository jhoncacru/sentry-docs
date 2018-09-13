```python
import sentry_sdk

def before_breadcrumb(crumb, hint):
    if 'log_record' in hint:
        crumb['data']['filename'] = hint['log_record'].filename
    return crumb

sentry_sdk.init(before_breadcrumb=before_breadcrumb)
```

For information about which hints are available see [hints in python]({% link _documentation/platforms/python/index.md %}#hints).