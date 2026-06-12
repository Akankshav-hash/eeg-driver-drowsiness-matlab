% testModel.m
% Validate the saved model on new signals/files

addpath('functions');

% 1) Load the deployable model
S = load('models/trainedModel.mat');   % loads finalModel, featureNames
finalModel   = S.finalModel;
featureNames = S.featureNames;

% Helper to classify one signal and print result
function [majLabel, perEpoch] = classifyOne(signal, fs, finalModel, featureNames)
    [Xepochs, ~] = preprocessEEG(signal, fs, 2, 0.5);
    T = extractFeatures(Xepochs, fs);
    % Reorder features exactly like training
    T = T(:, featureNames);
    X = T{:,:};

    [perEpoch, ~] = predict(finalModel, X);   % per-epoch labels (0/1)
    % Majority vote across epochs
    majLabel = mode(perEpoch);

    % Print quick summary
    n0 = sum(perEpoch==0); n1 = sum(perEpoch==1);
    fprintf('Per-epoch votes: Alert=%d, Drowsy=%d  -> Majority=%s\n', ...
        n0, n1, ternary(majLabel==0,'Alert','Drowsy'));
end

% tiny ternary helper
function out = ternary(cond, a, b)
    if cond, out = a; else, out = b; end
end

%% ------------------------------------------------------------------------
% A) Synthetic tests
fs = 256; t = 0:1/fs:10;

% A1) ALERT-like (10 Hz)
sig_alert   = sin(2*pi*10*t)' + 0.1*randn(length(t),1);
fprintf('\n=== Synthetic ALERT test ===\n');
[classAlert, perEpochA] = classifyOne(sig_alert, fs, finalModel, featureNames);

% A2) DROWSY-like (6 Hz)
sig_drowsy  = sin(2*pi*6*t)'  + 0.1*randn(length(t),1);
fprintf('\n=== Synthetic DROWSY test ===\n');
[classDrowsy, perEpochD] = classifyOne(sig_drowsy, fs, finalModel, featureNames);

%% ------------------------------------------------------------------------
% B) File-based tests (your saved files)
fprintf('\n=== File test: functions/data/alert.mat ===\n');
D = load('functions/data/alert.mat');   % expects variables: signal, fs
[classFileA, perEpochFA] = classifyOne(D.signal, D.fs, finalModel, featureNames);

fprintf('\n=== File test: functions/data/drowsy.mat ===\n');
D = load('functions/data/drowsy.mat');
[classFileD, perEpochFD] = classifyOne(D.signal, D.fs, finalModel, featureNames);

addpath('functions');

% 1) Load dataset again
fileList = { 'functions/data/alert.mat', 'functions/data/drowsy.mat' };
labels   = [0, 1];   % 0 = alert, 1 = drowsy

fs = 256;
DS = buildDataset(fileList, labels, fs, 2, 0.5);

X = DS{:,1:end-1};
Y = DS.Label;

% 2) Load trained model
S = load('models/trainedModel.mat');
finalModel   = S.finalModel;
featureNames = S.featureNames;

% Reorder features as during training
X = DS{:, featureNames};

% 3) Predict
Ypred = predict(finalModel, X);

% 4) Confusion Matrix
figure;
cm = confusionchart(Y, Ypred);
cm.Title = 'Confusion Matrix for Drowsiness Detection';
cm.RowSummary = 'row-normalized';
cm.ColumnSummary = 'column-normalized';

% 5) Accuracy
acc = mean(Y == Ypred);

% 6) Precision, Recall, F1
tp = sum((Y == 1) & (Ypred == 1));
tn = sum((Y == 0) & (Ypred == 0));
fp = sum((Y == 0) & (Ypred == 1));
fn = sum((Y == 1) & (Ypred == 0));

precision = tp / (tp + fp + eps);
recall    = tp / (tp + fn + eps);
f1        = 2 * (precision * recall) / (precision + recall + eps);

% 7) Print results
fprintf('\n--- Performance Metrics ---\n');
fprintf('Accuracy  = %.2f%%\n', acc*100);
fprintf('Precision = %.2f\n', precision);
fprintf('Recall    = %.2f\n', recall);
fprintf('F1-score  = %.2f\n', f1);
