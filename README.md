# CodeSample
Code style sample

```csharp
using UnityEngine;
using PlayerSystemDDX;
using System.Collections;
using UnityEngine.Localization;
using UnityEngine.SceneManagement;

namespace Games.TrueOrFalse
{
    [RequireComponent(typeof(FactContainer))]
    public class TrueOrFalseGamePlay : GamePlay<TrueOrFalsePlayer>
    {
        [Header("Info Panels")]
        [SerializeField] private InfoPanel _questionPanel;
        [SerializeField] private InfoPanel _questionNumberPanel;
        [SerializeField] private InfoPanel _preparePanel;
        [SerializeField] private InfoPanel _finishPanel;
        [SerializeField] private InfoPanel _roundResultPanel;

        [Header("Colors")]
        [SerializeField] private Color _trueColor;
        [SerializeField] private Color _falseColor;

        [Header("Localized Strings")]
        [SerializeField] private LocalizedString _prepareString;
        [SerializeField] private LocalizedString _intrigaString;
        [SerializeField] private LocalizedString _trueString;
        [SerializeField] private LocalizedString _falseString;

        [Header("Timings")]
        [SerializeField][Min(0)] private float _prepareTime = 3f;
        [SerializeField] [Min(0)] private float _intrigaTime = 3f;
        [SerializeField] [Min(0)] private float _answerTime = 3f;        
        [SerializeField] [Min(0)] private float _showingResultTime = 3f; 
        [SerializeField][Min(0)] private float _toNextSceneAwaitTime = 3f;

        [Header("On finish scene")]
        [SerializeField] private string _exitSceneName = "Main";

        private FactContainer _factContainer;
        private FactData _roundData;
        private TypeFact? _prevPlayerAnswer;

        protected override void Awake()
        {
            base.Awake();
            _factContainer = GetComponent<FactContainer>();
            _factContainer.Initialize();
        }

        protected override void OnLevelStarted()
        {
            base.OnLevelStarted();
            StartCoroutine(ShowPrepare());
        }

        protected override void OnRoundStarted()
        {
            base.OnRoundStarted();
            Debug.Log("Starting round...");

            _roundData = _factContainer.GetValue();

            _leftPlayer.ResetIsReady();
            _rightPlayer.ResetIsReady();

            _questionPanel.ShowText(_roundData.Fact.GetLocalizedString());
            _questionNumberPanel.ShowText(_roundData.Id.ToString());

            _prevPlayerAnswer = null;

            _leftPlayer.OnReady += OnPlayerReady;
            _rightPlayer.OnReady += OnPlayerReady;
        }

        protected override void OnLevelFinished()
        {
            base.OnLevelFinished();
            SaveScore();

            _finishPanel.SetVisibility(true);
            _finishPanel.ShowText(GetWinner().CommandName);

            StartCoroutine(AwaitAndMoveToNextScene());

        }


        private void OnPlayerReady(TrueOrFalsePlayer player, TypeFact answer)
        {
            Debug.Log($"{player.gameObject.name} select {answer}");

            player.OnReady -= OnPlayerReady;
            if (_prevPlayerAnswer == null)
            {
                _prevPlayerAnswer = answer;
            }
            else
            {
                var prevPlayer = _leftPlayer == player ? _rightPlayer : _leftPlayer;
                StartCoroutine(ShowResult(prevPlayer, _prevPlayerAnswer, player, answer));
            }
        }

        private IEnumerator ShowPrepare()
        {
            _preparePanel.SetVisibility(true);
            _preparePanel.ShowText(_prepareString.GetLocalizedString());
            yield return new WaitForSecondsRealtime(_prepareTime);
            _preparePanel.SetVisibility(false);
            NextRound();
        }

        private IEnumerator ShowResult(TrueOrFalsePlayer prevPlayer,
            TypeFact? prevPlayerAnswer, TrueOrFalsePlayer player, TypeFact playerAnswer)
        {
            yield return ShowIntrigue();

            if (prevPlayerAnswer == _roundData.Type) RaisePlayerScore(prevPlayer);
            if (playerAnswer == _roundData.Type) RaisePlayerScore(player);

            yield return ShowAnswer();

            yield return new WaitForSecondsRealtime(_showingResultTime);

            FinishRound();
            NextRound();
        }

        private IEnumerator ShowIntrigue()
        {
            _roundResultPanel.SetVisibility(true);
            _roundResultPanel.ShowText(_intrigaString.GetLocalizedString());
            yield return new WaitForSecondsRealtime(_intrigaTime);
        }

        private IEnumerator ShowAnswer()
        {
            var resultString = _roundData.Type == TypeFact.True ? _trueString : _falseString;
            _roundResultPanel.ShowText(resultString.GetLocalizedString(), _roundData.Type == TypeFact.True ? _trueColor : _falseColor);
            yield return new WaitForSecondsRealtime(_answerTime);
            _roundResultPanel.SetVisibility(false);
        }

        private IEnumerator AwaitAndMoveToNextScene()
        {
            yield return new WaitForSecondsRealtime(_toNextSceneAwaitTime);
            SceneManager.LoadScene(_exitSceneName);
        }
    }
}
