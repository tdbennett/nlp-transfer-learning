---
jupyter:
  jupytext:
    text_representation:
      extension: .md
      format_name: markdown
      format_version: '1.1'
      jupytext_version: 1.2.1
  kernelspec:
    display_name: Python 3
    language: python
    name: python3
---

# Based on our custom MIMIC language model, train a classifier

Make sure mimic_nlp_lm has been run first and sucessfully completed. That notebook builds the language model that allows classificiation to occur efficiently.

## Create a classifier to predict Length of Stay (LOS)

Steps:
1. Load clinical Notes
1. Load Admissions data
1. Calculate Length of Stay
1. Join Admissions data with Notes data (on HADM_ID) - Columns needed for classifier: LOS, TEXT

Would also be nice to see a graphical summary of LOS.

```python
from fastai.text import *
from sklearn.model_selection import train_test_split
import glob
import gc
import altair as alt
```

Setup filenames and paths

```python
# pandas doesn't understand ~, so provide full path
base_path = Path.home() / 'mimic'

# files used during processing - all aggregated here
admissions_file = base_path/'ADMISSIONS.csv'
notes_file = base_path/'NOTEEVENTS.csv'

class_file = 'los_cl_data.pickle'
notes_pickle_file = base_path/'noteevents.pickle'
lm_file = 'mimic_lm.pickle' # actual file is at base_path/lm_file but due to fastai function, have to pass file name separately
init_model_file = base_path/'los_cl_head'
cycles_file = base_path/'los_cl_num_iterations.pickle'
enc_file = 'mimic_fine_tuned_enc'
ft_file = 'los_cl_fine_tuned_'
freeze_two = base_path/'los_cl_freeze_two'
freeze_three = base_path/'los_cl_freeze_three'

training_history_file = 'los_cl_history'
```

Setup parameters for models

```python
# original data set too large to work with in reasonable time due to limted GPU resources
pct_data_sample = 0.1
# how much to hold out for validation
valid_pct = 0.2
# for repeatability - different seed than used with language model
seed = 1776
# for classifier, on unfrozen/full network training
# batch size of 128 GPU uses ?? GB RAM
# batch size of 96 GPU uses 22 GB RAM
# batch size of 48 GPU uses GB RAM
bs=96
```

```python
# if this doesn't free memory, can restart Python kernel.
# if that still doesn't work, try OS items mentioned here: https://docs.fast.ai/dev/gpu.html
def release_mem():
    gc.collect()
    torch.cuda.empty_cache()
release_mem()
```

```python
orig_df = pd.DataFrame()
if os.path.isfile(notes_pickle_file):
    print('Loading noteevent pickle file')
    orig_df = pd.read_pickle(notes_pickle_file)
    print(orig_df.shape)
else:
    print('Could not find noteevent pickle file; creating it')
    # run this the first time to covert CSV to Pickle file
    orig_df = pd.read_csv(notes_file, low_memory=False, memory_map=True)
    orig_df.to_pickle(notes_pickle_file)
```

Since seed is different, this should be quite different than the language model dataset.

Should I show details on how many records are in language model dataset?

```python
notes_df = orig_df[orig_df.HADM_ID.notnull()].copy()
notes_df.HADM_ID = notes_df.HADM_ID.astype(int)
notes_df.shape
```

```python
notes_df.head()
```

```python
print('Loading ADMISSIONS.csv')
a_orig = pd.read_csv(admissions_file, low_memory=False, memory_map=True)
a_orig.shape
```

```python
a_df = a_orig[['HADM_ID', 'ADMITTIME', 'DISCHTIME']].copy()
#pd.to_datetime('2014-04-09T152959.999993-0500', utc=True)
# passing format just to make sure conversion doesn't mess something up
a_df['admittime'] = pd.to_datetime(a_df.ADMITTIME, format='%Y-%m-%d %H:%M:%S')
a_df['dischtime'] = pd.to_datetime(a_df.DISCHTIME, format='%Y-%m-%d %H:%M:%S')
a_df['los'] = (a_df['dischtime'] - a_df['admittime']).astype('timedelta64[D]')
# can't use a float in neural network
a_df['los'] = a_df.los.astype(int)
# there are 98 admissions where length of stay is negative. change to 0
a_df.loc[a_df.los < 0, 'los'] = 0
a_df.head()
```

### Histogram of Length of Stay

```python
alt.Chart(a_df.head(4000)).mark_bar().encode(
    alt.X('los',
         bin=alt.BinParams(maxbins=50)),
    y='count()',
)
```

```python

```

```python
# make sure we only keep rows with notes
combined_df = pd.merge(a_df, notes_df, on='HADM_ID', how='right')

# passing format just to make sure conversion doesn't mess something up
combined_df['charttime'] = pd.to_datetime(combined_df.CHARTTIME, format='%Y-%m-%d %H:%M:%S')
combined_df['chartdate'] = pd.to_datetime(combined_df.CHARTDATE, format='%Y-%m-%d')
combined_df['admitdate'] = combined_df.admittime.dt.date
combined_df = combined_df[['HADM_ID', 'ROW_ID', 'admittime', 'admitdate', 'dischtime', 'los', 'chartdate', 'charttime', 'TEXT']]
combined_df.rename(columns={'HADM_ID': 'hadm_id', 'ROW_ID': 'row_id', 'TEXT': "text"}, inplace=True)
combined_df.shape
```

```python
# these should all be zero
print(combined_df[combined_df.los.isnull()].shape)
print(combined_df[combined_df.hadm_id.isnull()].shape)
print(combined_df[combined_df.text.isnull()].shape)
```

```python
combined_df.head()
```

```python
len(combined_df.hadm_id.unique())
```

### Use notes from day 1 of stay to predict LOS


    For each admission
        do they have notes on day 1 of stay

```python
combined_df.dtypes
```

```python
# Just look at one admission to see if this is the right filter criteria
combined_df[combined_df.hadm_id == 100006].sort_values(['chartdate', 'charttime'])
```

```python
# Just look at one admission to see if this is the right filter criteria
h = 100006
#for h in combined_df.HADM_ID.unique():
combined_df[(combined_df.hadm_id == h) & 
            (combined_df.charttime >= combined_df.admittime) &
            (combined_df.charttime < (combined_df.admittime + pd.Timedelta(hours=24)))
           ]
```

```python
# Just look at one admission to see if this is the right filter criteria
h = 100006
#for h in combined_df.HADM_ID.unique():
combined_df[(combined_df.hadm_id == h) & 
            (((combined_df.charttime >= combined_df.admittime) &
            (combined_df.charttime < (combined_df.admittime + pd.Timedelta(hours=24))))
             |
            (combined_df.chartdate == combined_df.admitdate))
           ]
```

### Histogram of note count each patient has

```python
alt.Chart(
    combined_df.groupby('hadm_id', as_index=False).text.count().sample(frac=.01, random_state=seed)
).mark_bar().encode(
    alt.X('text', bin=alt.BinParams(maxbins=50)),
    y='count()',
)
```

### Scatter plot of Number of Notes vs Length of Stay

```python
combined_df[['hadm_id', 'los']].drop_duplicates().shape
combined_df.groupby('hadm_id', as_index=False).text.count().shape  #58,361
```

```python
los_v_num_notes = pd.merge(combined_df[['hadm_id', 'los']].drop_duplicates(), 
          combined_df.groupby('hadm_id', as_index=False).text.count(),
          on='hadm_id')
los_v_num_notes.shape
```

```python
alt.Chart(los_v_num_notes.sample(frac=.08, random_state=seed)).mark_point().encode(
    x=alt.X('los', axis=alt.Axis(title='Length of Stay (Days)')),
    y=alt.Y('text', axis=alt.Axis(title='Number of Notes')))
```

### Build data set for LOS analysis

First, find rows of data related to first day of stay

Then, combine notes from first day into one text field

```python
# this is the slowest cell in pre-processing portion of the notebook 
fday = combined_df.groupby('hadm_id', as_index=False).apply(lambda g: g[
    (g.charttime >= g.admittime) & (g.charttime < (g.admittime + pd.Timedelta(hours=24)))
    |
    (g.chartdate == g.admitdate)
])
```

```python
fday.head()
```

```python
tmp = fday[['hadm_id', 'row_id']].reset_index(drop=True)
tmp['row_id'] = tmp['row_id'].astype(str)
tmp.dtypes
```

```python
tmp.head()
```

```python
combined_notes_row_ids = tmp.groupby(['hadm_id'], as_index=False).agg({
    'row_id': lambda x: ",".join(x)
})
combined_notes_row_ids.head()
```

```python
fday.head()
```

```python
combined_fday = fday.groupby(['hadm_id', 'los'], as_index=False).agg({
    'text': lambda x: "\n\n\n\n".join(x)
})
```

```python
combined_fday.head()
```

```python
combined_fday.shape
```

```python
len(combined_fday.los.unique())
```

```python
print(combined_fday.los.value_counts().head(10))
print(combined_fday.los.value_counts().tail(10))
```

```python
s = combined_fday.los.value_counts()
print('Number of records where length of stay is unique to that person:', len(s[s == 1]))
```

```python
# min
print('Min LOS:', combined_fday.los.min())
# max
print('Max LOS:', combined_fday.los.max())
# median
print('Median LOS:', combined_fday.los.median())
# mean
print('Mean LOS:', combined_fday.los.mean())
print('Mode LOS:', combined_fday.los.mode()[0]) # returns a series, just want the value
```

### Truncate LOS to max of 10

```python
trunc_fday = combined_fday.copy()
trunc_fday.loc[trunc_fday.los > 9, 'los'] = 10
trunc_fday.head()
```

```python
s = trunc_fday.los.value_counts()
len(s[s == 1])
```

```python
print(trunc_fday.los.value_counts().head(15))
```

```python
rowid_sample = combined_notes_row_ids.sample(frac=pct_data_sample, random_state=seed)
```

```python
df = trunc_fday.sample(frac=pct_data_sample, random_state=seed)
df.shape
```

```python
# should be 5535 - 100% overlap
len(set(rowid_sample.hadm_id.unique()) & set(df.hadm_id.unique()))
```

```python
len(df.hadm_id.unique())
```

```python
print('--------- stats on 10% random sample ---------')
print('Min LOS:', df.los.min())
print('Max LOS:', df.los.max())
print('Median LOS:', df.los.median())
print('Mean LOS:', df.los.mean())
print('Mode LOS:', df.los.mode()[0]) # returns a series, just want the value
```

```python
#s.apply(pd.Series).stack().reset_index(drop=True)

r = rowid_sample.groupby(['hadm_id'], as_index=False).agg({
    'row_id': lambda x: x.str.split(',')
})
row_ids = r.row_id.apply(pd.Series).stack().reset_index(drop=True)
```

```python
print(row_ids.shape)         # 33,914
print(len(row_ids.unique())) # 33,914
```

```python
# compare overlap between these notes and language model notes set
lm_df = orig_df.sample(frac=pct_data_sample, random_state=42)
print('rows in dataframe for NN:', len(row_ids.unique()))
print('rows in language model:', len(lm_df.ROW_ID.unique()))
print('row_ids in both:', len(set(row_ids.unique()) & set(lm_df.ROW_ID.unique())))
```

## Now for some Deep Learning

```python
# What if LOS was a string? Would accuracy, memory, or training time change?
# after some testing - no
# df_test = df.copy()
# df_test.los = df_test.los.apply(str)
# df_test.dtypes
```

```python
if os.path.isfile(base_path/lm_file):
    print('loading existing language model')
    lm = load_data(base_path, lm_file, bs=bs)
else:
    print('ERROR: language model file not found.')
```

#### This is a very CPU and RAM intensive process - no GPU involved

```python
filename = base_path/class_file
if os.path.isfile(filename):
    data_cl = load_data(base_path, class_file, bs=bs)
    print('loaded existing data bunch')
else:
    # do I need a vocab here? test with and without...
    data_cl = (TextList.from_df(df, base_path, cols='text', vocab=lm.vocab)
               #df has several columns; actual text is in column TEXT
               .split_by_rand_pct(valid_pct=valid_pct, seed=seed)
               #We randomly split and keep 20% for validation, set seed for repeatability
               .label_from_df(cols='los')
               .databunch(bs=bs))
    data_cl.save(filename)
    print('created new data bunch')
```

### Using weighted F1 to account for class imbalance

See https://scikit-learn.org/stable/modules/generated/sklearn.metrics.f1_score.html

```python
learn = text_classifier_learner(data_cl, AWD_LSTM, drop_mult=0.5, metrics=[accuracy, FBeta(average='weighted', beta=1)])
learn.load_encoder(enc_file)
```

```python
# with P100/P40 this takes maybe 5 minutes
# with 2017 macbook pro, intel core i7 3.1Ghz, this takes about 160 minutes
learn.lr_find()
```

```python
learn.recorder.plot()
```

Change learning rate based on results from the above plot.

Next several cells test various learning rates to find ideal learning rate

<!-- #region -->
```python
learn = text_classifier_learner(data_cl, AWD_LSTM, drop_mult=0.5, metrics=[accuracy, FBeta(average='weighted', beta=1)])
learn.load_encoder(enc_file)
learn.fit_one_cycle(1, 5e-1, moms=(0.8,0.7))
```

     epoch 	train_loss 	valid_loss 	accuracy 	f_beta 	time
        0	4.441254	2.457666	0.301716	0.174702	03:54
<!-- #endregion -->

<!-- #region -->
```python
learn = text_classifier_learner(data_cl, AWD_LSTM, drop_mult=0.5, metrics=[accuracy, FBeta(average='weighted', beta=1)])
learn.load_encoder(enc_file)
learn.fit_one_cycle(1, 3e-1, moms=(0.8,0.7))
```

     epoch 	train_loss 	valid_loss 	accuracy 	f_beta 	time
        0	2.329859	2.146445	0.319783	0.154967	03:44
<!-- #endregion -->

<!-- #region -->
```python
learn = text_classifier_learner(data_cl, AWD_LSTM, drop_mult=0.5, metrics=[accuracy, FBeta(average='weighted', beta=1)])
learn.load_encoder(enc_file)
learn.fit_one_cycle(1, 1e-1, moms=(0.8,0.7))
```

     epoch 	train_loss 	valid_loss 	accuracy 	f_beta 	time
        0	2.192971	2.083736	0.345077	0.211943	03:17
<!-- #endregion -->

<!-- #region -->
```python
learn = text_classifier_learner(data_cl, AWD_LSTM, drop_mult=0.5, metrics=[accuracy, FBeta(average='weighted', beta=1)])
learn.load_encoder(enc_file)
learn.fit_one_cycle(1, 5e-2, moms=(0.8,0.7))
```

     epoch 	train_loss 	valid_loss 	accuracy 	f_beta 	time
        0	2.183936	2.054368	0.359530	0.233471	03:56
<!-- #endregion -->

<!-- #region -->
```python
learn = text_classifier_learner(data_cl, AWD_LSTM, drop_mult=0.5, metrics=[accuracy, FBeta(average='weighted', beta=1)])
learn.load_encoder(enc_file)
learn.fit_one_cycle(1, 1e-2, moms=(0.8,0.7))
```

     epoch 	train_loss 	valid_loss 	accuracy 	f_beta 	time
        0	2.199809	2.068873	0.343270	0.217603	03:38
<!-- #endregion -->

### Train with selected learning rate

Results from `learn.fit_one_cycle(1, 1e-1, moms=(0.8,0.7))`

    Training new initial learner

    epoch 	train_loss 	valid_loss 	accuracy 	f_beta 	time
        0 	2.280483 	1.603017 	0.424571 	0.400360 	02:58

```python
if os.path.isfile(str(init_model_file) + '.pth'):
    learn.load(init_model_file)
    print('loaded initial learner')
else:
    print('Training new initial learner')
    learn.fit_one_cycle(1, 5e-2, moms=(0.8,0.7),
                       callbacks=[
                           callbacks.CSVLogger(learn, filename=training_history_file, append=True)
                       ])
    print('Saving new learner')
    learn.save(init_model_file)
    print('Finished generating new learner')
```

<!-- #region -->
### Results from the freeze_two learner


With `learn.fit_one_cycle(1, slice(5e-2/(2.6**4),5e-2), moms=(0.8,0.7))`

    epoch 	train_loss 	valid_loss 	accuracy 	f_beta 	time
        0	2.151176	2.060364	0.368564	0.266202	04:06
        
With `learn.fit_one_cycle(1, slice(1e-2/(2.6**4),1e-2), moms=(0.8,0.7))`

     epoch 	train_loss 	valid_loss 	accuracy 	f_beta 	time
        0	2.056317	2.013587	0.368564	0.245495	04:00
        
With `learn.fit_one_cycle(1, slice(5e-3/(2.6**4),5e-3), moms=(0.8,0.7))`

    epoch	train_loss	valid_loss	accuracy	f_beta	time
        0	2.335549	2.178197	0.236676	0.157695	04:05
        
With `learn.fit_one_cycle(1, slice(1e-3/(2.6**4),1e-3), moms=(0.8,0.7))`

     epoch 	train_loss 	valid_loss 	accuracy 	f_beta 	time
        0	2.529039	2.485589	0.075881	0.070762	03:26

<!-- #endregion -->
```python
if os.path.isfile(str(freeze_two) + '.pth'):
    learn.load(freeze_two)
    print('loaded freeze_two learner')
else:
    print('Training new freeze_two learner')
    learn.freeze_to(-2)
    learn.fit_one_cycle(1, slice(5e-2/(2.6**4),5e-2), moms=(0.8,0.7),
                       callbacks=[
                           callbacks.CSVLogger(learn, filename=training_history_file, append=True)
                       ])
    print('Saving new freeze_two learner')
    learn.save(freeze_two)
    print('Finished generating new freeze_two learner')
```

<!-- #region -->
```python
learn.load(freeze_two)
learn.freeze_to(-3)
learn.fit_one_cycle(1, slice(5e-2/(2.6**4),5e-2), moms=(0.8,0.7))
```

    epoch	train_loss	valid_loss	accuracy	f_beta	time
        0	2.010714	2.073426	0.373984	0.265069	04:21
<!-- #endregion -->

<!-- #region -->
```python
learn.load(freeze_two)
learn.freeze_to(-3)
learn.fit_one_cycle(1, slice(1e-2/(2.6**4),1e-2), moms=(0.8,0.7))
```

    epoch	train_loss	valid_loss	accuracy	f_beta	time
        0	1.941861	1.975329	0.369467	0.269504	03:50
<!-- #endregion -->

<!-- #region -->
```python
learn.load(freeze_two)
learn.freeze_to(-3)
learn.fit_one_cycle(1, slice(5e-3/(2.6**4),5e-3), moms=(0.8,0.7))
```

    epoch	train_loss	valid_loss	accuracy	f_beta	time
        0	1.958077	1.971695	0.367660	0.270906	03:51
<!-- #endregion -->

```python
if os.path.isfile(str(freeze_three) + '.pth'):
    learn.load(freeze_three)
    print('loaded freeze_three learner')
else:
    print('Training new freeze_three learner')
    learn.freeze_to(-3)
    learn.fit_one_cycle(1, slice(5e-2/(2.6**4),5e-2), moms=(0.8,0.7),
                       callbacks=[
                           callbacks.CSVLogger(learn, filename=training_history_file, append=True)
                       ])
    print('Saving new freeze_three learner')
    learn.save(freeze_three)
    print('Finished generating new freeze_three learner')
```

```python
learn.unfreeze()
```

```python
learn.lr_find()
learn.recorder.plot()
```

```python
release_mem()
```

<!-- #region -->
```python
learn.load(freeze_three)
learn.unfreeze()
learn.fit_one_cycle(1, slice(1e-4/(2.6**4),1e-4), moms=(0.8,0.7))
```

    epoch	train_loss	valid_loss	accuracy	f_beta	time
        0	2.070477	3.871025	0.356820	0.241513	04:13
<!-- #endregion -->

<!-- #region -->
```python
learn.load(freeze_three)
learn.unfreeze()
learn.fit_one_cycle(1, slice(5e-3/(2.6**4),5e-3), moms=(0.8,0.7))
```

    epoch	train_loss	valid_loss	accuracy	f_beta	time
        0	2.058974	2.119161	0.349593	0.247054	03:55
<!-- #endregion -->

<!-- #region -->
```python
learn.load(freeze_three)
learn.unfreeze()
learn.fit_one_cycle(1, slice(1e-3/(2.6**4),1e-3), moms=(0.8,0.7))
```

    epoch	train_loss	valid_loss	accuracy	f_beta	time
        0	2.070432	3.127100	0.354110	0.241749	04:20
<!-- #endregion -->

<!-- #region -->
```python
learn.load(freeze_three)
learn.unfreeze()
learn.fit_one_cycle(1, slice(5e-2/(2.6**4),5e-2), moms=(0.8,0.7))
```

    epoch	train_loss	valid_loss	accuracy	f_beta	time
        0	2.045097	2.424613	0.353207	0.249828	04:04
<!-- #endregion -->

<!-- #region -->
```python
learn.load(freeze_three)
learn.unfreeze()
learn.fit_one_cycle(1, slice(1e-2/(2.6**4),1e-2), moms=(0.8,0.7))
```

    epoch	train_loss	valid_loss	accuracy	f_beta	time
        0	2.039398	2.031859	0.357724	0.264459	04:29
<!-- #endregion -->

```python
learn = None
release_mem()
learn = text_classifier_learner(data_cl, AWD_LSTM, drop_mult=0.5, metrics=[accuracy, FBeta(average='weighted', beta=1)])
learn.load_encoder(enc_file)
learn.load(freeze_three)
learn.unfreeze()
```

```python
if os.path.isfile(cycles_file):
    with open(cycles_file, 'rb') as f:
        prev_cycles = pickle.load(f)
    print('This model has been trained for', prev_cycles, 'epochs already')  
else:
    prev_cycles = 0
    print('This model NOT been trained yet') 
```

```python
num_cycles = 3

file = ft_file + str(prev_cycles)
learner_file = base_path/file
callback_save_file = str(learner_file) + '_auto'

learn.fit_one_cycle(num_cycles, slice(1e-2/(2.6**4),1e-2), moms=(0.8,0.7),
                    callbacks=[
                        callbacks.SaveModelCallback(learn, every='epoch', monitor='accuracy', name=callback_save_file),
                        # CSVLogger only logs when num_cycles are complete
                        callbacks.CSVLogger(learn, filename=training_history_file, append=True)
                    ])
file = ft_file + str(prev_cycles + num_cycles)
learner_file = base_path/file
learn.save(learner_file)

with open(cycles_file, 'wb') as f:
    pickle.dump(num_cycles + prev_cycles, f)
release_mem()
```

```python
if os.path.isfile(cycles_file):
    with open(cycles_file, 'rb') as f:
        prev_cycles = pickle.load(f)
    print('This model has been trained for', prev_cycles, 'epochs already')  
else:
    prev_cycles = 0
    print('This model NOT been trained yet') 
```

```python
num_cycles = 4

file = ft_file + str(prev_cycles)
learner_file = base_path/file
callback_save_file = str(learner_file) + '_auto'

learn.fit_one_cycle(num_cycles, slice(1e-2/(2.6**4),1e-2), moms=(0.8,0.7),
                    callbacks=[
                        callbacks.SaveModelCallback(learn, every='epoch', monitor='accuracy', name=callback_save_file),
                        # CSVLogger only logs when num_cycles are complete
                        callbacks.CSVLogger(learn, filename=training_history_file, append=True)
                    ])
file = ft_file + str(prev_cycles + num_cycles)
learner_file = base_path/file
learn.save(learner_file)

with open(cycles_file, 'wb') as f:
    pickle.dump(num_cycles + prev_cycles, f)
release_mem()
```

```python

```
