::: {#4e7e904b .cell \_cell_guid=‘b1076dfc-b9ad-4769-8c92-a6c4dae69d19’ \_uuid=‘8f2839f25d086af736a60e9eeb907d3b93b6e0e5’ quarto-private-1=‘{“key”:“papermill”,“value”:{“duration”:0.999913,“end_time”:“2021-08-16T20:16:19.348363”,“exception”:false,“start_time”:“2021-08-16T20:16:18.348450”,“status”:“completed”}}’ tags=‘\[\]’ execution_count=1}

``` python
import numpy as np
import pandas as pd
from sklearn import model_selection
```

:::

``` python
df_train = pd.read_csv("../input/30-days-of-ml/train.csv")
```

``` python
df_train["kfold"] = -1
```

``` python
kf = model_selection.KFold(n_splits=5, shuffle=True, random_state=42)
for fold, (train_indicies, valid_indicies) in enumerate(kf.split(X=df_train)):
    df_train.loc[valid_indicies, "kfold"] = fold
```

``` python
df_train.to_csv("train_folds.csv", index=False)
```

Back to top
