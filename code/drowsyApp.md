classdef drowsyApp < matlab.apps.AppBase

   % Properties that correspond to app components
    properties (Access = public)
        UIFigure        matlab.ui.Figure
        ClearButton     matlab.ui.control.Button
        ResultLabel     matlab.ui.control.Label
        ClassifyButton  matlab.ui.control.Button
        LoadButton      matlab.ui.control.Button
        ResultAxes      matlab.ui.control.UIAxes
        RawEEGAxes      matlab.ui.control.UIAxes
    end

  % Callbacks that handle component events
    methods (Access = private)

   % Button pushed function: LoadButton
        function LoadButtonPushed(app, event)
            [file, path] = uigetfile('*.mat', 'Select EEG File');
            if isequal(file,0)
                return; % cancel
            end
            data = load(fullfile(path, file));

   % pick EEG
            if isfield(data,'signal')
                EEG = data.signal(:);
            else
                f = fieldnames(data);
                EEG = data.(f{1})(:);   % take first variable
            end

   % pick fs if available
            if isfield(data,'fs')
                fs = data.fs;
            else
                fs = 256; % default
            end

   % store in UIFigure appdata
            setappdata(app.UIFigure,'EEG',EEG);
            setappdata(app.UIFigure,'fs',fs);

  % plot
            t = (0:length(EEG)-1)/fs;
            plot(app.RawEEGAxes,t,EEG);
            title(app.RawEEGAxes,'Raw EEG Signal');
            xlabel(app.RawEEGAxes,'Time (s)');
            ylabel(app.RawEEGAxes,'Amplitude');

   app.ResultLabel.Text = 'Result: ---';
            cla(app.ResultAxes);
        end

  % Button pushed function: ClassifyButton
        function ClassifyButtonPushed(app, event)
         if ~isappdata(app.UIFigure,'EEG')
             uialert(app.UIFigure,'Load an EEG file first!','Error');
             return;
         end
         EEG = getappdata(app.UIFigure,'EEG');
         fs  = getappdata(app.UIFigure,'fs');

  % load model
         modelData = load('models/trainedModel.mat');
         finalModel = modelData.finalModel;
         featureNames = modelData.featureNames;

  % preprocess + features
         [Xepochs, ~] = preprocessEEG(EEG, fs, 2, 0.5);
         feats = extractFeatures(Xepochs, fs);

  % reorder features
         feats = feats(:,featureNames);
         X = feats{:,:};

   % predict
         [labels,~] = predict(finalModel,X);
         votesAlert = sum(labels==0);
         votesDrowsy = sum(labels==1);

  % plot votes
         n0 = sum(labels == 0);
         n1 = sum(labels == 1);
         bar(app.ResultAxes, [n0 n1]);
         title(app.ResultAxes, 'Classification Result');
         set(app.ResultAxes, 'XTickLabel', {'Alert','Drowsy'});
         ylabel(app.ResultAxes, 'Votes');
         % update label
         if votesAlert >= votesDrowsy
             app.ResultLabel.Text = 'Result: ALERT 😃';
         else
             app.ResultLabel.Text = 'Result: DROWSY 😴';
         end
        end

   % Button pushed function: ClearButton
        function ClearButtonPushed(app, event)
            cla(app.RawEEGAxes);
            cla(app.ResultAxes);

   title(app.RawEEGAxes,'Raw EEG Signal');
            xlabel(app.RawEEGAxes,'Time (s)');
            ylabel(app.RawEEGAxes,'Amplitude');

  title(app.ResultAxes,'Classification Result');
            xlabel(app.ResultAxes,'Class');
            ylabel(app.ResultAxes,'Votes');

  app.ResultLabel.Text = 'Result: ---';

   % remove stored data
             if isappdata(app.UIFigure,'EEG')
                 rmappdata(app.UIFigure,'EEG');
             end
             if isappdata(app.UIFigure,'fs')
                 rmappdata(app.UIFigure,'fs');
             end
        end
    end

  % Component initialization
    methods (Access = private)

  % Create UIFigure and components
        function createComponents(app)

  % Create UIFigure and hide until all components are created
            app.UIFigure = uifigure('Visible', 'off');
            app.UIFigure.Position = [100 100 640 480];
            app.UIFigure.Name = 'MATLAB App';

   % Create RawEEGAxes
            app.RawEEGAxes = uiaxes(app.UIFigure);
            title(app.RawEEGAxes, 'Raw EEG Signal')
            xlabel(app.RawEEGAxes, 'Time (s)')
            ylabel(app.RawEEGAxes, 'Amplitude')
            app.RawEEGAxes.Position = [32 187 290 185];

  % Create ResultAxes
            app.ResultAxes = uiaxes(app.UIFigure);
            title(app.ResultAxes, 'Classification Result')
            xlabel(app.ResultAxes, 'Class')
            ylabel(app.ResultAxes, 'Votes')
            app.ResultAxes.Position = [339 187 282 173];

  % Create LoadButton
            app.LoadButton = uibutton(app.UIFigure, 'push');
            app.LoadButton.ButtonPushedFcn = createCallbackFcn(app, @LoadButtonPushed, true);
            app.LoadButton.Position = [145 380 100 22];
            app.LoadButton.Text = 'Load EEG File';

  % Create ClassifyButton
            app.ClassifyButton = uibutton(app.UIFigure, 'push');
            app.ClassifyButton.ButtonPushedFcn = createCallbackFcn(app, @ClassifyButtonPushed, true);
            app.ClassifyButton.Position = [435 380 100 22];
            app.ClassifyButton.Text = 'Classify';

   % Create ResultLabel
            app.ResultLabel = uilabel(app.UIFigure);
            app.ResultLabel.HorizontalAlignment = 'center';
            app.ResultLabel.FontSize = 20;
            app.ResultLabel.FontWeight = 'bold';
            app.ResultLabel.Position = [250 140 250 30];
            app.ResultLabel.Text = 'Result: ---';

  % Create ClearButton
            app.ClearButton = uibutton(app.UIFigure, 'push');
            app.ClearButton.ButtonPushedFcn = createCallbackFcn(app, @ClearButtonPushed, true);
            app.ClearButton.Position = [290 380 100 22];
            app.ClearButton.Text = 'Clear';

  % Show the figure after all components are created
            app.UIFigure.Visible = 'on';
        end
    end

   % App creation and deletion
    methods (Access = public)

  % Construct app
        function app = drowsyApp

   % Create UIFigure and components
            createComponents(app)

   % Register the app with App Designer
            registerApp(app, app.UIFigure)

  if nargout == 0
                clear app
            end
        end

  % Code that executes before app deletion
        function delete(app)

  % Delete UIFigure when app is deleted
            delete(app.UIFigure)
        end
    end
end
