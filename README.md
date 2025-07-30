📊 YouTube Comments & Popularity Analysis
Project Type: SQL Analytics
Environment: SSMS (SQL Server Management Studio)
Objective: To explore the relationship between video popularity and the nature of user-generated comments, including sentiment. The goal is to generate actionable insights for content creators and platform strategists.

🧠 Project Overview
This project analyzes YouTube video-level statistics and associated comment data. Using SQL, we explore how user engagement (views, likes, comments) aligns with audience reactions (comment sentiment, comment volume, top-liked comments). The insights help identify what types of content drive engagement and how users emotionally respond to it.

📌 Business Questions & Analytical Focus

1. Which keywords are associated with the highest-performing videos?
Consider analyzing by views, likes, and comments.
Are there common patterns in topics or content types?

```
SELECT 
    Keyword,
    COUNT(*) AS VideoCount,
    SUM(CAST(Views AS FLOAT)) AS TotalViews,
    SUM(CAST(Likes AS FLOAT)) AS TotalLikes,
    SUM(CAST(Comments AS FLOAT)) AS TotalComments,
    (SUM(CAST(Views AS FLOAT)) + 10 * SUM(CAST(Likes AS FLOAT)) + 20 * SUM(CAST(Comments AS FLOAT))) AS PerformanceScore
FROM videos
WHERE Keyword IS NOT NULL AND Keyword <> ''
  AND ISNUMERIC(Views) = 1 AND ISNUMERIC(Likes) = 1 AND ISNUMERIC(Comments) = 1
GROUP BY Keyword
ORDER BY PerformanceScore DESC;

/*2. Do videos with high engagement (likes/views/comments) tend to have more positive, negative, or neutral comments?
Use average sentiment across comments per video.
Do high-performing videos attract more extreme sentiment?*/
WITH AvgSentimentPerVideo AS (
    SELECT 
        video_id,
        AVG(TRY_CAST(Sentiment AS FLOAT)) AS AvgSentiment
    FROM Comments
    WHERE TRY_CAST(Sentiment AS FLOAT) IS NOT NULL
    GROUP BY video_id
)
SELECT 
    V.Title,
    TRY_CAST(V.Views AS FLOAT) AS Views,
    TRY_CAST(V.Likes AS FLOAT) AS Likes,
    TRY_CAST(V.Comments AS FLOAT) AS CommentCount,
    S.AvgSentiment,
    TRY_CAST(V.Likes AS FLOAT) / NULLIF(TRY_CAST(V.Views AS FLOAT), 0) AS LikeToViewRatio,
    TRY_CAST(V.Comments AS FLOAT) / NULLIF(TRY_CAST(V.Views AS FLOAT), 0) AS CommentToViewRatio
FROM videos V
JOIN AvgSentimentPerVideo S ON V.video_id = S.video_id
WHERE TRY_CAST(V.Likes AS FLOAT) IS NOT NULL
  AND TRY_CAST(V.Views AS FLOAT) IS NOT NULL
  AND TRY_CAST(V.Comments AS FLOAT) IS NOT NULL
ORDER BY Views DESC;

/*3. What is the relationship between comment volume and comment sentiment?
Do videos with more comments have more polarized (0 or 2) reactions?
Are neutral comments more common on low-engagement videos?*/
WITH SentimentCounts AS (
    SELECT 
        video_id,
        SUM(CASE WHEN TRY_CAST(Sentiment AS INT) = 0 THEN 1 ELSE 0 END) AS NegativeCount,
        SUM(CASE WHEN TRY_CAST(Sentiment AS INT) = 1 THEN 1 ELSE 0 END) AS NeutralCount,
        SUM(CASE WHEN TRY_CAST(Sentiment AS INT) = 2 THEN 1 ELSE 0 END) AS PositiveCount,
        COUNT(*) AS TotalComments
    FROM comments
    WHERE TRY_CAST(Sentiment AS INT) IS NOT NULL
    GROUP BY video_id
)
SELECT 
    V.Title,
    V.Views,
    V.Comments AS CommentCount,
    SC.TotalComments,
    SC.NegativeCount,
    SC.NeutralCount,
    SC.PositiveCount,
    
    CAST(SC.NeutralCount AS FLOAT) / NULLIF(SC.TotalComments, 0) AS NeutralPct,
    CAST((SC.NegativeCount + SC.PositiveCount) AS FLOAT) / NULLIF(SC.TotalComments, 0) AS PolarizedPct
FROM SentimentCounts SC
JOIN videos V ON V.video_id = SC.video_id
WHERE TRY_CAST(V.Comments AS INT) IS NOT NULL
ORDER BY V.Comments DESC;

/*4. Which videos have the most liked comments, and what is the sentiment of those top comments?
Are the top-liked comments generally positive or critical?
Do they align with the video's popularity? */
WITH TopLikedComment AS (
    SELECT 
        video_id,
        Comment,
        Sentiment,
        Likes,
        ROW_NUMBER() OVER (PARTITION BY video_id ORDER BY TRY_CAST(Likes AS INT) DESC) AS rn
    FROM comments
    WHERE TRY_CAST(Likes AS INT) IS NOT NULL
      AND TRY_CAST(Sentiment AS INT) IS NOT NULL
)
SELECT 
    V.Title,
    V.Views,
    V.Likes AS VideoLikes,
    V.Comments AS TotalComments,
    
    C.Comment AS TopComment,
    C.Likes AS TopCommentLikes,
    C.Sentiment AS TopCommentSentiment
FROM TopLikedComment C
JOIN videos V ON V.video_id = C.video_id
WHERE C.rn = 1
ORDER BY TRY_CAST(C.Likes AS INT) DESC;

/*5. What proportion of videos have disabled comments or hidden like counts?
Do these videos still perform well in terms of views?
Is there a trend by keyword or publishing date?*/
WITH VideoFlags AS (
    SELECT
        video_id,
        Title,
        Keyword,
        published_at,
        TRY_CAST(Views AS FLOAT) AS Views,
        TRY_CAST(Likes AS FLOAT) AS Likes,
        TRY_CAST(Comments AS FLOAT) AS Comments,

        -- Flag: Comments disabled if Comments is NULL or 0
        CASE 
            WHEN TRY_CAST(Comments AS FLOAT) IS NULL OR TRY_CAST(Comments AS FLOAT) = 0 
                THEN 1 
            ELSE 0 
        END AS CommentsDisabled,

        -- Flag: Likes hidden if Likes is NULL or 0
        CASE 
            WHEN TRY_CAST(Likes AS FLOAT) IS NULL OR TRY_CAST(Likes AS FLOAT) = 0 
                THEN 1 
            ELSE 0 
        END AS LikesHidden
    FROM videos
)
SELECT
    COUNT(*) AS TotalVideos,
    SUM(CommentsDisabled) AS NumCommentsDisabled,
    SUM(LikesHidden) AS NumLikesHidden,

    CAST(SUM(CommentsDisabled) AS FLOAT) / COUNT(*) AS PctCommentsDisabled,
    CAST(SUM(LikesHidden) AS FLOAT) / COUNT(*) AS PctLikesHidden,

    -- Average views for each group
    AVG(CASE WHEN CommentsDisabled = 1 THEN Views END) AS AvgViews_CommentsDisabled,
    AVG(CASE WHEN CommentsDisabled = 0 THEN Views END) AS AvgViews_CommentsEnabled,

    AVG(CASE WHEN LikesHidden = 1 THEN Views END) AS AvgViews_LikesHidden,
    AVG(CASE WHEN LikesHidden = 0 THEN Views END) AS AvgViews_LikesVisible
FROM VideoFlags;

/*6. Are there keywords that consistently result in higher comment sentiment or more liked comments?
Can creators use this insight to drive more engagement?*/
SELECT 
    V.Keyword,
    COUNT(DISTINCT V.video_id) AS NumVideos,
    COUNT(C.Comment) AS NumComments,
    
    -- Sentiment: avg per keyword
    AVG(TRY_CAST(C.Sentiment AS FLOAT)) AS AvgSentiment,

    -- Average likes on comments under this keyword
    AVG(TRY_CAST(C.Likes AS FLOAT)) AS AvgCommentLikes,

    -- Engagement: comments per video
    COUNT(C.Comment) * 1.0 / NULLIF(COUNT(DISTINCT V.video_id), 0) AS CommentsPerVideo
FROM videos V
JOIN comments C ON V.video_id = C.video_id
WHERE TRY_CAST(C.Sentiment AS FLOAT) IS NOT NULL
  AND TRY_CAST(C.Likes AS FLOAT) IS NOT NULL
GROUP BY V.Keyword
ORDER BY AvgSentiment DESC;

/*7. How does the publication date relate to performance?
Are newer videos trending better or worse than older ones?
Does sentiment vary over time?*/
WITH MonthlyStats AS (
    SELECT 
        FORMAT(CAST(V.published_at AS DATETIME), 'yyyy-MM') AS PublishMonth,
        AVG(TRY_CAST(V.Views AS FLOAT)) AS AvgViews,
        AVG(TRY_CAST(V.Likes AS FLOAT)) AS AvgLikes,
        AVG(TRY_CAST(V.Comments AS FLOAT)) AS AvgComments,
        AVG(TRY_CAST(C.Sentiment AS FLOAT)) AS AvgSentiment,
        COUNT(DISTINCT V.video_id) AS TotalVideos,
        COUNT(C.Comment) AS TotalComments
    FROM videos V
    LEFT JOIN comments C ON V.video_id = C.video_id
    WHERE 
        TRY_CAST(V.Views AS FLOAT) IS NOT NULL AND
        TRY_CAST(V.Likes AS FLOAT) IS NOT NULL AND
        TRY_CAST(V.Comments AS FLOAT) IS NOT NULL AND
        TRY_CAST(C.Sentiment AS FLOAT) IS NOT NULL
    GROUP BY FORMAT(CAST(V.published_at AS DATETIME), 'yyyy-MM')
)
SELECT *
FROM MonthlyStats
ORDER BY PublishMonth;

