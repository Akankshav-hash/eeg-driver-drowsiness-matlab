function [Xepochs, tIdx] = preprocessEEG(signal, fs, winSec, overlap)
% Preprocess EEG: bandpass 0.5-45 Hz, notch 50 Hz, zscore, and epoching.

if isvector(signal)
    signal = signal(:); % N x 1
end
[N,C] = size(signal);

% Bandpass filter
[b,a] = butter(4, [0.5 45]/(fs/2), 'bandpass');
sig = filtfilt(b,a,signal);

% 50 Hz notch
wo = 50/(fs/2); bw = wo/35;
[bn,an] = iirnotch(wo,bw);
sig = filtfilt(bn,an,sig);

% Normalize
sig = (sig - mean(sig,1)) ./ max(std(sig,[],1),eps);

% Epoching
W = round(winSec*fs);          % samples per window
S = round(W * (1-overlap));    % step size
starts = 1:S:(N-W+1);
E = numel(starts);
Xepochs = zeros(E,W,C);
for e = 1:E
    idx = starts(e):(starts(e)+W-1);
    Xepochs(e,:,:) = sig(idx,:);
end
tIdx = starts(:);
end
