% trainModel.m (deployable)
% Train EEG drowsiness classifier and save a final model

addpath('functions');   % make sure helper functions are visible

% ------------------------------------------------------------
% 1) Build dataset (you kept data inside functions/data/)
% ------------------------------------------------------------
fileList = { 'functions/data/alert.mat', 'functions/data/drowsy.mat' };
labels   = [0, 1];   % 0 = alert, 1 = drowsy

fs = 256;  % sampling frequency
DS = buildDataset(fileList, labels, fs, 2, 0.5);  % 2s window, 50% overlap

% Split into features and labels
X = DS{:,1:end-1};
Y = DS.Label;

% ------------------------------------------------------------
% 2) Cross-validated accuracy (for info only)
% ------------------------------------------------------------
cvMdl = fitcsvm(X, Y, ...
    'KernelFunction','rbf', ...
    'Standardize',true, ...
    'ClassNames',[0 1]);
cv5 = crossval(cvMdl,'KFold',5);
acc = 1 - kfoldLoss(cv5);
fprintf('Cross-validated Accuracy: %.2f%%\n', acc*100);

% ------------------------------------------------------------
% 3) Train FINAL model on ALL data (this is what we will save)
% ------------------------------------------------------------
finalModel = fitcsvm(X, Y, ...
    'KernelFunction','rbf', ...
    'Standardize',true, ...
    'ClassNames',[0 1]);

featureNames = DS.Properties.VariableNames(1:end-1);

% ------------------------------------------------------------
% 4) Save deployable model + metadata
% ------------------------------------------------------------
if ~exist('models','dir'), mkdir('models'); end
save('models/trainedModel.mat','finalModel','featureNames');

disp('✅ Training complete. Saved models/trainedModel.mat (finalModel + featureNames).');
