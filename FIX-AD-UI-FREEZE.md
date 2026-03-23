# Fix: UI freeze after rewarded ad on iOS

## Bug
After watching a rewarded ad (chest opening), the entire UI becomes unresponsive.
Only the Share button works (because it uses native iOS share sheet, bypassing Unity UI).
User must force-quit the app.

## Root Cause
Unity Ads callbacks (`OnUnityAdsShowComplete`, `OnUnityAdsShowFailure`) can be invoked
on a background thread on iOS. Any UI modification done in these callbacks is silently
ignored, leaving the blocking overlay / loading state permanently active.

## Fix — 3 changes required

### 1. Ensure all ad callbacks dispatch to the main thread

In every script that implements `IUnityAdsShowListener`, wrap callback logic
with a main-thread dispatcher. The simplest approach:

```csharp
// Add this field
private bool _adCompleted = false;
private bool _adFailed = false;
private UnityAdsShowCompletionState _completionState;

public void OnUnityAdsShowComplete(string adUnitId, UnityAdsShowCompletionState state)
{
    // Do NOT touch UI here — may be on background thread on iOS
    _completionState = state;
    _adCompleted = true;
}

public void OnUnityAdsShowFailure(string adUnitId, UnityAdsShowError error, string message)
{
    Debug.LogError($"Ad show failed: {error} - {message}");
    _adFailed = true;
}

private void Update()
{
    if (_adCompleted)
    {
        _adCompleted = false;
        HandleAdCompleted(_completionState);
    }
    if (_adFailed)
    {
        _adFailed = false;
        HandleAdFailed();
    }
}

private void HandleAdCompleted(UnityAdsShowCompletionState state)
{
    // NOW safe to modify UI (we're on the main thread)
    RemoveBlockingOverlay();
    if (state == UnityAdsShowCompletionState.COMPLETED)
    {
        GiveReward();
    }
}

private void HandleAdFailed()
{
    // Also remove blocking overlay on failure!
    RemoveBlockingOverlay();
}
```

### 2. Always remove the blocking overlay — even on failure/skip

Make sure `RemoveBlockingOverlay()` is called in ALL paths:
- Ad completed successfully
- Ad failed to show
- Ad was skipped by user
- Ad load failed (don't even show the overlay if the ad isn't loaded)

```csharp
private void RemoveBlockingOverlay()
{
    if (loadingOverlay != null)
        loadingOverlay.SetActive(false);

    if (blockingCanvasGroup != null)
    {
        blockingCanvasGroup.blocksRaycasts = false;
        blockingCanvasGroup.interactable = true;
    }

    // Re-enable EventSystem if you disabled it
    if (EventSystem.current != null)
        EventSystem.current.enabled = true;

    // Restore time scale if you paused it
    Time.timeScale = 1f;
}
```

### 3. Add a safety timeout as fallback

In case the callback never fires at all (rare but possible on iOS):

```csharp
private float _adShowTime;
private bool _waitingForAd = false;
private const float AD_TIMEOUT = 30f; // seconds

public void ShowRewardedAd()
{
    if (Advertisement.isShowing) return;

    ShowBlockingOverlay();
    _waitingForAd = true;
    _adShowTime = Time.realtimeSinceStartup;
    Advertisement.Show(adUnitId, this);
}

private void Update()
{
    // ... (callback dispatching from step 1)

    // Safety timeout
    if (_waitingForAd &&
        Time.realtimeSinceStartup - _adShowTime > AD_TIMEOUT)
    {
        Debug.LogWarning("Ad callback timeout — forcing UI unlock");
        _waitingForAd = false;
        RemoveBlockingOverlay();
    }
}
```

## Where to apply

Apply these changes in every script that shows a rewarded ad:
- The chest opening script (workshop/atelier)
- The end-of-dive chest opening script
- Any other rewarded ad entry point

## Testing

1. Open a chest via ad on iOS (both in workshop and end-of-dive)
2. Verify all buttons are clickable after the ad
3. Test with airplane mode (ad fail path)
4. Test closing the ad early (skip path)
