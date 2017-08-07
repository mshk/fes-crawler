const client = require('cheerio-httpcli')
const SpotifyWebApi = require('spotify-web-api-node');
const fs = require('fs');
const axios = require('axios');
const MAX_BATCH_REQUEST = 50

desc('fetch rsr artists')
task('fetch-rsr-2017-artists', (format = 'json', max = 3) => {
  client.fetch('http://rsr.wess.co.jp/2017/artists/lineup/', (err, $, res) => {

    let promises = [];
    let delay = 0;
    let data = {};

    $('.tab_area_blc').each(function (idx) {
      const stageNameElem = $(this).children('h4')
      const stageName = stageNameElem.children('img').attr('alt')
      const stageDate = stageNameElem.text().replace(/[\n\r]/g, '').replace(/  /g, '')

      if (data[stageDate] == null) {
        data[stageDate] = {}
      }

      if (data[stageDate][stageName] == null) {
        data[stageDate][stageName] = []
      }

      $(this).children('ul').each(function (idx) {
        $(this).children('li').each(function (idx) {
          const artistName = $(this).children('a').children('i').text().replace(/\n/g, '').replace(/  /g, '')

          data[stageDate][stageName].push(artistName)
        });
      });
    })

    console.log(JSON.stringify(data))
  })
});

desc('get spotify authcode')
task('get-spotify-authcode-url', () => {
  // credentials are optional
  let spotifyApi = new SpotifyWebApi({
    clientId: process.env.SPOTIFY_CLIENT_ID,
    clientSecret: process.env.SPOTIFY_CLIENT_SECRET,
    redirectUri: 'http://127.0.0.1/callback'
  });

  scopes = ['playlist-modify-public', 'playlist-modify-private'];

  const authUrl = spotifyApi.createAuthorizeURL(scopes, '');

  console.log(authUrl);
});

desc('get spotify token')
task('get-spotify-authtoken', (authCode) => {
  // credentials are optional
  const spotifyApi = new SpotifyWebApi({
    clientId: process.env.SPOTIFY_CLIENT_ID,
    clientSecret: process.env.SPOTIFY_CLIENT_SECRET,
    redirectUri: 'http://127.0.0.1/callback'
  });

  const code = authCode || process.env.SPOTIFY_AUTH_CODE

  spotifyApi.authorizationCodeGrant(code)
    .then(function (data) {
      console.log('The token expires in ' + data.body['expires_in']);
      console.log('The access token is ' + data.body['access_token']);
      console.log('The refresh token is ' + data.body['refresh_token']);

      process.env['SPOTIFY_ACCESS_TOKEN'] = data.body['access_token'];
      process.env['SPOTIFY_REFRESH_TOKEN'] = data.body['refresh_token'];
    }, function (err) {
      console.log('Something went wrong!', err);
    });
});


desc('search spotify for artists')
task('search-spotify', (configFilePath) => {
  // credentials are optional
  const spotifyApi = new SpotifyWebApi({
    clientId: process.env.SPOTIFY_CLIENT_ID,
    clientSecret: process.env.SPOTIFY_CLIENT_SECRET,
    redirectUri: 'http://127.0.0.1/callback'
  });

  const timetable = JSON.parse(fs.readFileSync(configFilePath, 'utf8'));

  // Set the access token on the API object to use it in later calls
  spotifyApi.setAccessToken(process.env.SPOTIFY_AUTH_TOKEN);
  spotifyApi.setRefreshToken(process.env.SPOTIFY_REFRESH_TOKEN);

  let tracks = {};
  let promises = []
  let delay = 0
  let start = Date.now()

  for (let date in timetable) {
    for (let stage in timetable[date]) {
      timetable[date][stage].forEach((artist) => {
        if (tracks[date] == null) { tracks[date] = {}; }
        if (tracks[date][stage] == null) { tracks[date][stage] = {}; }
        if (tracks[date][stage][artist] == null) { tracks[date][stage][artist] = []; }

        promises.push(new Promise((resolve, reject) => {
          setTimeout(() => {
            console.error(Date.now() - start);

            spotifyApi.searchTracks(`artist:${artist}`)
              .then(function (data) {
                let prevTrackName = null;

                data.body.tracks.items.forEach((item) => {
                  if (item.name == prevTrackName) return;

                  tracks[date][stage][artist].push({
                    name: item.name,
                    album_name: item.album.name,
                    album_images: item.album.images,
                    url: item.external_urls.spotify,
                    id: item.id,
                    uri: item.uri,
                    preview_url: item.preview_url
                  });

                  prevTrackName = item.name
                });
                resolve();
              }, function (err) {
                console.error('Something went wrong!', err);
                reject();
              });
          }, delay);
        }))
        delay += 400;
      })
    }
  }

  Promise.all(promises)
    .then(() => {
      console.log(JSON.stringify(tracks));
    })
    .catch((error) => {
      console.error('Promise.all error: ', error);
      console.log(JSON.stringify(tracks));
    })
});

desc('search spotify for artists')
task('update-spotify-playlist', (trackFilePath) => {
  const playlistId = process.env.SPOTIFY_PLAYLIST_ID;
  const userId = process.env.SPOTIFY_USER_ID;

  // credentials are optional
  const spotifyApi = new SpotifyWebApi({
    clientId: process.env.SPOTIFY_CLIENT_ID,
    clientSecret: process.env.SPOTIFY_CLIENT_SECRET,
    redirectUri: 'http://127.0.0.1/callback'
  });

  const tracks = JSON.parse(fs.readFileSync(trackFilePath, 'utf8'));

  let trackUris = []
  for (let date in tracks) {
    for (let stage in tracks[date]) {
      for (let artist in tracks[date][stage]) {
        tracks[date][stage][artist].slice(0, 3).forEach((track) => {
          if (track.uri) {
            trackUris.push(track.uri)
          }
        })
      };
    }
  }

  const options = {};

  spotifyApi.setAccessToken(process.env.SPOTIFY_ACCESS_TOKEN)

  let promises = []
  let delay = 0
  let start = Date.now()

  while (trackUris.length > 0) {
    const spliced = trackUris.splice(0, 50)
    promises.push(new Promise((resolve, reject) => {
      setTimeout(() => {
        console.error(Date.now() - start)
        spotifyApi.addTracksToPlaylist(userId, playlistId, spliced)
          .then(() => {
            resolve();
          })
          .catch(() => {
            reject();
          })
      }, delay);
    }))
    delay += 400
  }

  Promise.all(promises)
    .then(function (data) {
      console.log('Added tracks to the playlist!', data);
    }).catch(function (err) {
      console.log('Something went wrong!', err);
    });

});


desc('showartists')
task('show-artists', (timetableFilePath) => {
  const timetable = JSON.parse(fs.readFileSync(timetableFilePath, 'utf8'));

  for (let date in timetable) {
    for (let stage in timetable[date]) {
      timetable[date][stage].forEach((artist) => {
        console.log(artist);
      });
    }
  }

});

desc('showartists')
task('show-artists-in-tracks', (trackFilePath) => {
  const tracks = JSON.parse(fs.readFileSync(trackFilePath, 'utf8'));

  for (let date in tracks) {
    for (let stage in tracks[date]) {
      for (let artist in tracks[date][stage]) {
        console.log(artist)
      };
    }
  }
});

desc('showartists')
task('show-empty-artists-in-tracks', (trackFilePath) => {
  const tracks = JSON.parse(fs.readFileSync(trackFilePath, 'utf8'));

  for (let date in tracks) {
    for (let stage in tracks[date]) {
      for (let artist in tracks[date][stage]) {
        if (tracks[date][stage][artist].length == 0) {
          console.log(artist)
        }
      };
    }
  }
});



