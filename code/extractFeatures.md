function T = extractFeatures(Xepochs, fs)
% Extract features from EEG epochs
% Input:
%   Xepochs - [E × W × C] epochs (E = epochs, W = samples, C = channels)
%   fs      - sampling frequency
% Output:
%   T       - table of features

[E,W,C] = size(Xepochs);
features = [];

for c = 1:C
    mu = mean(Xepochs(:,:,c),2);
    sd = std(Xepochs(:,:,c),0,2);
    
   PSD settings (safe version)
    nfft = min(2^nextpow2(W), W);     % ensure nfft ≤ W
    win = hamming(max(16, floor(W/4))); % use quarter of window, at least 16
    noverlap = floor(numel(win)/2);   % 50% overlap
    
  bp_bands = [0.5 4; 4 8; 8 13; 13 30];  % delta, theta, alpha, beta
    bp_power = zeros(E, size(bp_bands,1));
    
   spec_entropy = zeros(E,1);
    
  for e = 1:E
        xc = squeeze(Xepochs(e,:,c));
        [Pxx,F] = pwelch(xc, win, noverlap, nfft, fs);
        Pxx = Pxx / sum(Pxx + eps); % normalize PSD
        
        
  for b = 1:size(bp_bands,1)
            idx = F>=bp_bands(b,1) & F<bp_bands(b,2);
            bp_power(e,b) = sum(Pxx(idx));
        end
        
        % Spectral entropy
  spec_entropy(e) = -sum(Pxx .* log(Pxx + eps));
    end
    
    % Combine features for this channel
   feats = table(mu, sd, bp_power(:,1), bp_power(:,2), bp_power(:,3), bp_power(:,4), spec_entropy, ...
        'VariableNames',{['ch' num2str(c) '_mu'], ['ch' num2str(c) '_sd'], ...
                         ['ch' num2str(c) '_bp_delta'], ['ch' num2str(c) '_bp_theta'], ...
                         ['ch' num2str(c) '_bp_alpha'], ['ch' num2str(c) '_bp_beta'], ...
                         ['ch' num2str(c) '_spec_entropy']});
    features = [features feats];
end

T = features;
end
